"""
Sri Dakshina Dravida Dharma Peetham — All-in-one Discord bot ("Sanatana Margadarshak")
Slash-command only (/dharma, /shastra, /learn, /ai, /mod, /level, /config, /info, /tools, /ticket ...)
AI chat (Groq, free) · Deep web search · Web scraper · Leveling · Tickets · Logging · Automod

⚠️  Your Discord token and Groq key are typed into the CONFIG section below, so this file is a
    secret. Never upload it, paste it publicly, or commit it to a public repo. If a token leaks,
    regenerate it in the Discord Developer Portal.

Run:   pip install -r requirements.txt   then   python main.py
Data (levels, warnings, banned words, settings) lives in bot_data.db next to this file.
"""

from __future__ import annotations  # keeps `X | None` type hints safe on older Python versions

import os
import re
import ast
import time
import random
import asyncio
import operator
import platform
import textwrap
import datetime
import socket
import ipaddress
import urllib.parse
from typing import Optional
from concurrent.futures import ThreadPoolExecutor

import discord
from discord import app_commands
from discord.ext import commands

import requests
from bs4 import BeautifulSoup

import aiosqlite
from openai import OpenAI

# ═══════════════════════════════════════════════════════════════════════════
# CONFIG — fill in the two keys once. No environment variables needed.
# ═══════════════════════════════════════════════════════════════════════════

DISCORD_TOKEN = os.getenv("DISCORD_TOKEN", "").strip()
GROQ_API_KEY = os.getenv("GROQ_API_KEY", "").strip()

SAFFRON = 0xFF9933
DB_PATH = "bot_data.db"
AI_MODEL = "llama-3.3-70b-versatile"        # main model (free on Groq)
AI_FALLBACK_MODEL = "llama-3.1-8b-instant"  # automatic fallback if the main one errors / is rate-limited

# --- Your server's IDs (all can be changed later with /config ...) ---
WELCOME_CHANNEL_ID = 1552955693903122482    # welcome embeds
GOODBYE_CHANNEL_ID = 1552955693903122482    # goodbye embeds (separate setting — change here or with /config setgoodbyechannel)
LOG_CHANNEL_ID = 1535527378850283551        # staff log: deletes, joins, leaves, level-ups, mod actions, automod, tickets
VERIFY_CHANNEL_ID = 1542120365856129105     # where the verification ticket panel is posted

LEVEL_ROLES = {                              # level -> role id (500 words per level)
    1: 1552954940643745883,
    2: 1552955104099835965,
    3: 1552955192272617482,                  # 1500 words — give this role Attach Files + Embed Links in Server Settings > Roles
    4: 1552955297105186866,
    5: 1552955390713528350,
}
WORDS_PER_LEVEL = 500
XP_COOLDOWN_SECONDS = 10
PEETHAM = "Sri Dakshina Dravida Dharma Peetham"

# ═══════════════════════════════════════════════════════════════════════════
# Persona / prompts
# ═══════════════════════════════════════════════════════════════════════════

CHAT_SYSTEM = textwrap.dedent(
    f"""
    You are Sanatana Margadarshak, a friendly, knowledgeable and deeply traditional Hindu guide who hangs out
    in the Discord server of {PEETHAM}. You chat like a real person, not like a customer-support bot.

    HOW YOU TALK
    - Casual, warm, natural, like a friend who happens to know the shastras. Short replies by default
      (1-3 sentences). Mirror the user's energy and language (English, Hindi, Hinglish, Tamil, Bengali...).
      Examples: "hello" -> "yo wassup 🙏"; "kya haal hai" -> "badhiya bhai, tu bata"; "thanks" -> "anytime!".
    - No headings, no bullet lists, no bold-spam unless the user asks for a structured answer.
      Never say "As an AI..." or "How can I assist you today?".
    - When the topic turns spiritual or shastric (dharma, karma, rituals, scriptures, philosophy), shift into a
      calm, respectful panditji tone and give a fuller answer, still conversational and not a lecture dump.
    - Use traditional greetings sparingly and naturally (Hari Om, Namo Narayanaya, Jai Shree Ram), not in every message.
    - If someone sincerely asks whether you are a bot or an AI, tell them honestly that you are a bot.

    TRADITIONAL OUTLOOK
    - You follow the traditional orthodox understanding: the Vedas as pramana, the Itihasa-Puranas, Agamas,
      Divya Prabandham and the teachings of the Purvacharyas, with deep respect for Dakshina Bharata temple traditions.
    - On varna: the traditional (orthodox Dharmashastra) position is that varna is by janma (birth), within
      varnashrama-dharma and svadharma. Explain this faithfully as the traditional position when asked. If someone asks
      about other interpretations (for example guna-karma readings of Gita 4.13), you may acknowledge honestly that some
      teachers read it differently, without arguing or mocking anyone.
    - Never demean, insult or incite hatred against any person, community, caste or religion. Respect for all beings
      (Gita 5.18) is part of the tradition. Politely decline to produce abuse or hate.
    - You describe traditions. You do not give medical, legal or financial advice, and you do not predict the future.

    VERSE INTEGRITY (very important)
    - NEVER write Sanskrit or Tamil verses from memory and never invent verse numbers or chapter references.
    - You may quote a verse ONLY if it appears under VERIFIED VERSES below. Copy it exactly, character for character,
      with the reference given.
    - If the user wants a verse and none is provided, tell them to use /dharma search or /dharma verse for verified text,
      and explain the idea in your own words. Name a scripture only if you are certain about it.
    """
).strip()

ASK_EXTRA = (
    "The user used a command to ask a real question. Give a genuinely helpful, accurate, well-organised answer "
    "(roughly 120-250 words) in flowing conversational prose, still plain text with no headings."
)

# ═══════════════════════════════════════════════════════════════════════════
# Welcome / goodbye / ticket text
# ═══════════════════════════════════════════════════════════════════════════

WELCOME_BODY = textwrap.dedent(
    """
    "धर्मो रक्षति रक्षितः"
    *(Dharma protects those who protect Dharma.)*

    Hari Om and a warm, traditional welcome to Sri Dakshina Dravida Dharma Peetham! We are deeply honored by your arrival (Atithi Devo Bhava).

    This sacred digital Peetham (seat of learning) is dedicated to preserving, discussing, and celebrating the rich spiritual heritage, philosophy, temple traditions, and sacred literature of Dakshina Bharata (South India) and Sanatana Dharma.

    🙏 **Guidelines for Seekers (Jijnasus):**
    **Sanskriti & Maryada:** Maintain a serene, respectful, and dignified tone at all times.
    **Shastric Discussion:** Engage in constructive discussions on the Vedas, Agamas, Divya Prabandham, Stotram, and philosophy.
    **Mutual Respect:** Respect all acharyas, traditions, and fellow devotees.

    Check:- <#1545376513711677531>
    Check:- <#1542120365856129105>
    Check:- <#1545376689415000104>

    May the divine blessings of the Almighty and our Purvacharyas guide you on your journey toward knowledge and Dharma.

    Namo Narayanaya | Hari Om Tat Sat
    """
).strip()

GOODBYE_BODY = textwrap.dedent(
    """
    "लोकाः समस्ताः सुखिनो भवन्तु"
    *(May all beings everywhere be happy and free.)*

    Hari Om, noble seeker. {mention} has taken leave from Sri Dakshina Dravida Dharma Peetham.

    We express our gratitude for the time you spent within this sacred assembly (Sabha). May the divine grace of the Almighty and the eternal wisdom of Sanatana Dharma illuminate your path wherever your journey leads.

    You will always remain welcome to return to this seat of learning.

    Hari Om Tat Sat | Namo Narayanaya 🙏
    """
).strip()

TICKET_QUESTIONS = textwrap.dedent(
    """
    Please answer the following questions here in this ticket so our team may verify you:

    **1.** What is your religion?
    **2.** Why do you want to join the server?
    **3.** Do you agree to the server rules and Discord's Terms of Service?
    """
).strip()

# ═══════════════════════════════════════════════════════════════════════════
# VERIFIED VERSE LIBRARY
# Every verse the bot ever shows or is allowed to quote comes from here, never from AI memory.
# ═══════════════════════════════════════════════════════════════════════════


def V(ref: str, sa: str, en: str, tags: str) -> dict:
    return {"ref": ref, "sa": sa.strip(), "en": en, "tags": set(tags.split())}


VERSES: dict[str, dict] = {
    "bg-2-47": V("Bhagavad Gita 2.47", "कर्मण्येवाधिकारस्ते मा फलेषु कदाचन ।\nमा कर्मफलहेतुर्भूर्मा ते सङ्गोऽस्त्वकर्मणि ॥",
                 "You have a right to action alone, never to its fruits. Let not the fruit of action be your motive, nor let there be attachment to inaction.",
                 "karma action duty work detachment fruits phala niskama"),
    "bg-2-48": V("Bhagavad Gita 2.48", "योगस्थः कुरु कर्माणि सङ्गं त्यक्त्वा धनञ्जय ।\nसिद्ध्यसिद्ध्योः समो भूत्वा समत्वं योग उच्यते ॥",
                 "Established in yoga, perform actions, abandoning attachment, O Dhananjaya, remaining even-minded in success and failure. Evenness of mind is called yoga.",
                 "yoga equanimity success failure samatva balance"),
    "bg-2-13": V("Bhagavad Gita 2.13", "देहिनोऽस्मिन्यथा देहे कौमारं यौवनं जरा ।\nतथा देहान्तरप्राप्तिर्धीरस्तत्र न मुह्यति ॥",
                 "As the embodied soul passes in this body through childhood, youth and old age, so does it pass to another body. The wise are not deluded by this.",
                 "atma soul death rebirth body samsara"),
    "bg-2-14": V("Bhagavad Gita 2.14", "मात्रास्पर्शास्तु कौन्तेय शीतोष्णसुखदुःखदाः ।\nआगमापायिनोऽनित्यास्तांस्तितिक्षस्व भारत ॥",
                 "Contacts of the senses with their objects give cold and heat, pleasure and pain. They come and go and are impermanent; endure them, O Bharata.",
                 "suffering pain pleasure patience tolerance impermanence hardship"),
    "bg-2-20": V("Bhagavad Gita 2.20", "न जायते म्रियते वा कदाचिन्नायं भूत्वा भविता वा न भूयः ।\nअजो नित्यः शाश्वतोऽयं पुराणो न हन्यते हन्यमाने शरीरे ॥",
                 "The Self is never born and never dies; having once been, it never ceases to be. Unborn, eternal, everlasting and ancient, it is not slain when the body is slain.",
                 "atma soul death eternal immortality self"),
    "bg-2-22": V("Bhagavad Gita 2.22", "वासांसि जीर्णानि यथा विहाय नवानि गृह्णाति नरोऽपराणि ।\nतथा शरीराणि विहाय जीर्णान्यन्यानि संयाति नवानि देही ॥",
                 "As a person casts off worn-out garments and puts on new ones, so the embodied soul casts off worn-out bodies and enters new ones.",
                 "atma soul death rebirth body samsara reincarnation"),
    "bg-2-23": V("Bhagavad Gita 2.23", "नैनं छिन्दन्ति शस्त्राणि नैनं दहति पावकः ।\nन चैनं क्लेदयन्त्यापो न शोषयति मारुतः ॥",
                 "Weapons cannot cut the Self, fire cannot burn it, water cannot wet it, and the wind cannot dry it.",
                 "atma soul eternal self indestructible"),
    "bg-2-62": V("Bhagavad Gita 2.62", "ध्यायतो विषयान्पुंसः सङ्गस्तेषूपजायते ।\nसङ्गात्सञ्जायते कामः कामात्क्रोधोऽभिजायते ॥",
                 "Dwelling on sense objects, a person develops attachment to them. From attachment springs desire, and from desire anger arises.",
                 "anger desire attachment mind kama krodha senses"),
    "bg-2-63": V("Bhagavad Gita 2.63", "क्रोधाद्भवति सम्मोहः सम्मोहात्स्मृतिविभ्रमः ।\nस्मृतिभ्रंशाद् बुद्धिनाशो बुद्धिनाशात्प्रणश्यति ॥",
                 "From anger comes delusion; from delusion, loss of memory; from loss of memory, destruction of intellect; and from the destruction of intellect, one perishes.",
                 "anger delusion mind intellect krodha fall"),
    "bg-3-21": V("Bhagavad Gita 3.21", "यद्यदाचरति श्रेष्ठस्तत्तदेवेतरो जनः ।\nस यत्प्रमाणं कुरुते लोकस्तदनुवर्तते ॥",
                 "Whatever a great person does, others follow. Whatever standard they set, the world follows.",
                 "leader example conduct role model society"),
    "bg-3-35": V("Bhagavad Gita 3.35", "श्रेयान्स्वधर्मो विगुणः परधर्मात्स्वनुष्ठितात् ।\nस्वधर्मे निधनं श्रेयः परधर्मो भयावहः ॥",
                 "Better is one's own dharma, though imperfect, than another's dharma well performed. Death in one's own dharma is better; the dharma of another brings fear.",
                 "svadharma dharma duty varna"),
    "bg-4-7": V("Bhagavad Gita 4.7", "यदा यदा हि धर्मस्य ग्लानिर्भवति भारत ।\nअभ्युत्थानमधर्मस्य तदात्मानं सृजाम्यहम् ॥",
                "Whenever there is a decline of dharma and a rise of adharma, O Bharata, then I manifest Myself.",
                "avatar dharma incarnation krishna adharma decline"),
    "bg-4-8": V("Bhagavad Gita 4.8", "परित्राणाय साधूनां विनाशाय च दुष्कृताम् ।\nधर्मसंस्थापनार्थाय सम्भवामि युगे युगे ॥",
                "For the protection of the righteous, the destruction of evildoers and the establishment of dharma, I appear age after age.",
                "avatar dharma incarnation krishna protection"),
    "bg-4-13": V("Bhagavad Gita 4.13", "चातुर्वर्ण्यं मया सृष्टं गुणकर्मविभागशः ।\nतस्य कर्तारमपि मां विद्ध्यकर्तारमव्ययम् ॥",
                 "The fourfold varna was created by Me according to the division of guna and karma. Though I am its creator, know Me to be the non-doer, the imperishable.",
                 "varna caste chaturvarnya guna"),
    "bg-4-38": V("Bhagavad Gita 4.38", "न हि ज्ञानेन सदृशं पवित्रमिह विद्यते ।\nतत्स्वयं योगसंसिद्धः कालेनात्मनि विन्दति ॥",
                 "Truly, nothing in this world is as purifying as knowledge. One perfected in yoga finds it within the Self in due time.",
                 "knowledge jnana purity wisdom learning"),
    "bg-5-18": V("Bhagavad Gita 5.18", "विद्याविनयसम्पन्ने ब्राह्मणे गवि हस्तिनि ।\nशुनि चैव श्वपाके च पण्डिताः समदर्शिनः ॥",
                 "The wise see with equal vision a learned and humble brahmana, a cow, an elephant, a dog and an outcaste.",
                 "equality samadarshi humility respect varna wise"),
    "bg-6-5": V("Bhagavad Gita 6.5", "उद्धरेदात्मनात्मानं नात्मानमवसादयेत् ।\nआत्मैव ह्यात्मनो बन्धुरात्मैव रिपुरात्मनः ॥",
                "Lift yourself by yourself; do not let yourself sink. The self alone is the friend of the self, and the self alone is its enemy.",
                "self mind discipline motivation willpower confidence"),
    "bg-9-22": V("Bhagavad Gita 9.22", "अनन्याश्चिन्तयन्तो मां ये जनाः पर्युपासते ।\nतेषां नित्याभियुक्तानां योगक्षेमं वहाम्यहम् ॥",
                 "To those who worship Me with single-minded devotion, ever steadfast, I carry what they lack and preserve what they have.",
                 "bhakti devotion protection surrender yogakshema trust"),
    "bg-9-26": V("Bhagavad Gita 9.26", "पत्रं पुष्पं फलं तोयं यो मे भक्त्या प्रयच्छति ।\nतदहं भक्त्युपहृतमश्नामि प्रयतात्मनः ॥",
                 "Whoever offers Me with devotion a leaf, a flower, a fruit or water, that loving offering of the pure-hearted I accept.",
                 "bhakti offering puja worship simplicity devotion"),
    "bg-10-8": V("Bhagavad Gita 10.8", "अहं सर्वस्य प्रभवो मत्तः सर्वं प्रवर्तते ।\nइति मत्वा भजन्ते मां बुधा भावसमन्विताः ॥",
                 "I am the source of all; everything proceeds from Me. Knowing this, the wise worship Me with loving devotion.",
                 "creation source bhakti god krishna"),
    "bg-12-13": V("Bhagavad Gita 12.13", "अद्वेष्टा सर्वभूतानां मैत्रः करुण एव च ।\nनिर्ममो निरहङ्कारः समदुःखसुखः क्षमी ॥",
                  "One who hates no being, who is friendly and compassionate, free from possessiveness and ego, equal in pain and pleasure, and forgiving (is dear to Me).",
                  "compassion devotee qualities kindness forgiveness friendship character"),
    "bg-16-21": V("Bhagavad Gita 16.21", "त्रिविधं नरकस्येदं द्वारं नाशनमात्मनः ।\nकामः क्रोधस्तथा लोभस्तस्मादेतत्त्रयं त्यजेत् ॥",
                  "Threefold is this gate to hell, destructive of the self: desire, anger and greed. Therefore one should abandon these three.",
                  "kama krodha lobha anger greed desire hell vices"),
    "bg-17-15": V("Bhagavad Gita 17.15", "अनुद्वेगकरं वाक्यं सत्यं प्रियहितं च यत् ।\nस्वाध्यायाभ्यसनं चैव वाङ्मयं तप उच्यते ॥",
                  "Speech that causes no distress, that is truthful, pleasant and beneficial, and the regular study of the scriptures: this is called austerity of speech.",
                  "speech truth tapas words communication"),
    "bg-18-41": V("Bhagavad Gita 18.41", "ब्राह्मणक्षत्रियविशां शूद्राणां च परन्तप ।\nकर्माणि प्रविभक्तानि स्वभावप्रभवैर्गुणैः ॥",
                  "Of brahmanas, kshatriyas, vaishyas and shudras, O Parantapa, the duties are distributed according to the gunas born of their own nature.",
                  "varna caste svabhava duty"),
    "bg-18-42": V("Bhagavad Gita 18.42", "शमो दमस्तपः शौचं क्षान्तिरार्जवमेव च ।\nज्ञानं विज्ञानमास्तिक्यं ब्रह्मकर्म स्वभावजम् ॥",
                  "Tranquility, self-control, austerity, purity, forbearance, uprightness, knowledge, wisdom and faith: these are the natural duties of a brahmana.",
                  "varna brahmana duty qualities"),
    "bg-18-47": V("Bhagavad Gita 18.47", "श्रेयान्स्वधर्मो विगुणः परधर्मात्स्वनुष्ठितात् ।\nस्वभावनियतं कर्म कुर्वन्नाप्नोति किल्बिषम् ॥",
                  "Better is one's own dharma, though imperfect, than another's well performed. Performing the duty prescribed by one's own nature, one incurs no sin.",
                  "svadharma duty varna"),
    "bg-18-66": V("Bhagavad Gita 18.66", "सर्वधर्मान्परित्यज्य मामेकं शरणं व्रज ।\nअहं त्वां सर्वपापेभ्यो मोक्षयिष्यामि मा शुचः ॥",
                  "Abandoning all dharmas, take refuge in Me alone. I shall free you from all sins; do not grieve.",
                  "surrender sharanagati prapatti moksha refuge charama grief fear"),
    "bg-18-78": V("Bhagavad Gita 18.78", "यत्र योगेश्वरः कृष्णो यत्र पार्थो धनुर्धरः ।\nतत्र श्रीर्विजयो भूतिर्ध्रुवा नीतिर्मतिर्मम ॥",
                  "Wherever there is Krishna, the Lord of yoga, and Arjuna, the archer, there are fortune, victory, prosperity and firm righteousness. Such is my conviction.",
                  "victory krishna success fortune"),
    "isha": V("Isha Upanishad 1", "ईशावास्यमिदं सर्वं यत्किञ्च जगत्यां जगत् ।\nतेन त्यक्तेन भुञ्जीथा मा गृधः कस्यस्विद्धनम् ॥",
              "All this, whatever moves in this moving world, is pervaded by the Lord. Enjoy through renunciation; do not covet anyone's wealth.",
              "isha ishvara renunciation greed upanishad wealth contentment"),
    "asatoma": V("Brihadaranyaka Upanishad 1.3.28", "ॐ असतो मा सद्गमय ।\nतमसो मा ज्योतिर्गमय ।\nमृत्योर्मा अमृतं गमय ॥\nॐ शान्तिः शान्तिः शान्तिः ॥",
                 "Lead me from untruth to truth, from darkness to light, from death to immortality. Om, peace, peace, peace.",
                 "truth light darkness immortality shanti prayer asato"),
    "purnam": V("Isha Upanishad (Shanti Mantra) / Brihadaranyaka 5.1.1", "ॐ पूर्णमदः पूर्णमिदं पूर्णात्पूर्णमुदच्यते ।\nपूर्णस्य पूर्णमादाय पूर्णमेवावशिष्यते ॥\nॐ शान्तिः शान्तिः शान्तिः ॥",
                "That is whole, this is whole; from the whole the whole arises. Taking the whole from the whole, the whole alone remains.",
                "purna whole brahman infinite shanti upanishad"),
    "matru": V("Taittiriya Upanishad 1.11.2", "मातृदेवो भव ।\nपितृदेवो भव ।\nआचार्यदेवो भव ।\nअतिथिदेवो भव ।",
               "Let your mother be a god to you; let your father be a god; let your teacher be a god; let your guest be a god.",
               "guest atithi mother father teacher respect hospitality welcome parents guru"),
    "satyam-vada": V("Taittiriya Upanishad 1.11.1", "सत्यं वद । धर्मं चर ।\nस्वाध्यायान्मा प्रमदः ।",
                     "Speak the truth. Follow dharma. Do not neglect your svadhyaya (study of the scriptures).",
                     "truth satya dharma study svadhyaya convocation"),
    "saha": V("Shanti Mantra (Katha / Taittiriya Upanishad)", "ॐ सह नाववतु ।\nसह नौ भुनक्तु ।\nसह वीर्यं करवावहै ।\nतेजस्वि नावधीतमस्तु मा विद्विषावहै ॥\nॐ शान्तिः शान्तिः शान्तिः ॥",
              "May He protect us both (teacher and student); may He nourish us both; may we work together with vigour; may our study be illuminating; may we never hate one another. Om, peace, peace, peace.",
              "shanti teacher student study peace prayer learning"),
    "mundaka": V("Mundaka Upanishad 3.1.6", "सत्यमेव जयते नानृतं सत्येन पन्था विततो देवयानः ।",
                 "Truth alone triumphs, not falsehood. By truth the path to the gods is laid out.",
                 "truth satya victory honesty"),
    "katha": V("Katha Upanishad 1.3.14", "उत्तिष्ठत जाग्रत प्राप्य वरान्निबोधत ।",
               "Arise, awake, and having reached the excellent teachers, understand (the Self).",
               "arise awake knowledge guru motivation learning"),
    "gayatri": V("Gayatri Mantra, Rigveda 3.62.10", "ॐ भूर्भुवः स्वः ।\nतत्सवितुर्वरेण्यं भर्गो देवस्य धीमहि ।\nधियो यो नः प्रचोदयात् ॥",
                 "We meditate on the adorable radiance of the divine Savitr; may He inspire our intellects.",
                 "gayatri savitr sun intellect meditation sandhya"),
    "mrityunjaya": V("Maha Mrityunjaya Mantra, Rigveda 7.59.12", "ॐ त्र्यम्बकं यजामहे सुगन्धिं पुष्टिवर्धनम् ।\nउर्वारुकमिव बन्धनान्मृत्योर्मुक्षीय मामृतात् ॥",
                     "We worship the three-eyed Lord, fragrant and nourishing. As a ripe cucumber is freed from its stem, may we be liberated from death, not from immortality.",
                     "shiva mrityunjaya death health healing protection"),
    "vasudhaiva": V("Maha Upanishad 6.71", "अयं निजः परो वेति गणना लघुचेतसाम् ।\nउदारचरितानां तु वसुधैव कुटुम्बकम् ॥",
                    "'This one is mine, that one a stranger' is the thinking of the small-minded. For those of noble conduct, the whole world is a family.",
                    "family world unity brotherhood inclusion universal"),
    "manu-dharma": V("Manusmriti 8.15", "धर्म एव हतो हन्ति धर्मो रक्षति रक्षितः ।\nतस्माद्धर्मो न हन्तव्यो मा नो धर्मो हतोऽवधीत् ॥",
                     "Dharma, when destroyed, destroys; dharma, when protected, protects. Therefore dharma must not be destroyed, lest destroyed dharma destroy us.",
                     "dharma protect protection rakshati"),
    "sarve": V("Traditional Shanti Verse", "सर्वे भवन्तु सुखिनः सर्वे सन्तु निरामयाः ।\nसर्वे भद्राणि पश्यन्तु मा कश्चिद्दुःखभाग्भवेत् ॥",
               "May all be happy; may all be free from illness; may all see auspiciousness; may no one suffer.",
               "peace shanti happiness wellbeing prayer welfare all"),
    "guru": V("Guru Gita (Skanda Purana)", "गुरुर्ब्रह्मा गुरुर्विष्णुः गुरुर्देवो महेश्वरः ।\nगुरुः साक्षात् परं ब्रह्म तस्मै श्रीगुरवे नमः ॥",
              "The Guru is Brahma, the Guru is Vishnu, the Guru is Maheshwara. The Guru is verily the Supreme Brahman; salutations to that revered Guru.",
              "guru teacher acharya respect student"),
    "shantakaram": V("Vishnu Dhyana Shloka (traditional)", "शान्ताकारं भुजगशयनं पद्मनाभं सुरेशं\nविश्वाधारं गगनसदृशं मेघवर्णं शुभाङ्गम् ।\nलक्ष्मीकान्तं कमलनयनं योगिभिर्ध्यानगम्यं\nवन्दे विष्णुं भवभयहरं सर्वलोकैकनाथम् ॥",
                   "I bow to Vishnu: of peaceful form, reclining on the serpent, with a lotus in His navel, Lord of the gods, support of the universe, vast as the sky, cloud-hued, of auspicious limbs, consort of Lakshmi, lotus-eyed, attained by yogis through meditation, remover of the fear of samsara, the one Lord of all worlds.",
                   "vishnu narayana peace dhyana meditation"),
    "shuklambara": V("Traditional Dhyana Shloka (before beginning any task)", "शुक्लाम्बरधरं विष्णुं शशिवर्णं चतुर्भुजम् ।\nप्रसन्नवदनं ध्यायेत् सर्वविघ्नोपशान्तये ॥",
                     "One should meditate on the four-armed, moon-hued Lord clad in white, with a cheerful face, for the removal of all obstacles.",
                     "obstacles vighna ganesha vishnu beginning start"),
    "karpura": V("Shiva Dhyana Shloka (traditional)", "कर्पूरगौरं करुणावतारं संसारसारं भुजगेन्द्रहारम् ।\nसदा वसन्तं हृदयारविन्दे भवं भवानीसहितं नमामि ॥",
                 "I bow to Bhava (Shiva), camphor-white, an incarnation of compassion, the essence of existence, adorned with the king of serpents, who ever dwells in the lotus of the heart together with Bhavani.",
                 "shiva mahadeva compassion dhyana arati"),
    "vakratunda": V("Ganesha Shloka (traditional)", "वक्रतुण्ड महाकाय सूर्यकोटि समप्रभ ।\nनिर्विघ्नं कुरु मे देव सर्वकार्येषु सर्वदा ॥",
                    "O Lord of the curved trunk and mighty body, radiant as a million suns, make all my undertakings free of obstacles, always.",
                    "ganesha ganapati obstacles vighna beginning success"),
    "manojavam": V("Hanuman Shloka (traditional)", "मनोजवं मारुततुल्यवेगं जितेन्द्रियं बुद्धिमतां वरिष्ठम् ।\nवातात्मजं वानरयूथमुख्यं श्रीरामदूतं शिरसा नमामि ॥",
                   "I bow my head to Sri Rama's messenger: swift as the mind, fast as the wind, master of the senses, foremost among the wise, son of the Wind God, chief of the vanara host.",
                   "hanuman strength courage devotion rama"),
    "sarvamangala": V("Devi Mahatmya (Durga Saptashati), Narayani Stuti", "सर्वमङ्गलमाङ्गल्ये शिवे सर्वार्थसाधिके ।\nशरण्ये त्र्यम्बके गौरि नारायणि नमोऽस्तु ते ॥",
                      "O Narayani, auspicious of all that is auspicious, consort of Shiva, fulfiller of all aims, giver of refuge, three-eyed Gauri, salutations to you.",
                      "devi durga shakti mother protection navaratri"),
    "saraswati": V("Saraswati Pranama (traditional)", "सरस्वति नमस्तुभ्यं वरदे कामरूपिणि ।\nविद्यारम्भं करिष्यामि सिद्धिर्भवतु मे सदा ॥",
                   "Salutations to you, Saraswati, giver of boons, who assumes forms at will. I begin my studies; may success always be mine.",
                   "saraswati vidya learning study education exams knowledge"),
    "krishna": V("Krishna Pranama (traditional)", "कृष्णाय वासुदेवाय हरये परमात्मने ।\nप्रणतः क्लेशनाशाय गोविन्दाय नमो नमः ॥",
                 "Salutations again and again to Krishna, Vasudeva, Hari, the Supreme Self, Govinda, the destroyer of the sorrows of those who bow to Him.",
                 "krishna govinda vasudeva hari sorrow refuge"),
    "rama-nama": V("Sri Vishnu Sahasranama (closing verses)", "श्रीराम राम रामेति रमे रामे मनोरमे ।\nसहस्रनाम तत्तुल्यं रामनाम वरानने ॥",
                   "Rama, Rama, Rama: thus I rejoice in the delightful Rama. O fair-faced one, the name Rama equals a thousand names.",
                   "rama nama name japa sahasranama"),
    "ramo-dharma": V("Valmiki Ramayana, Aranya Kanda", "रामो विग्रहवान् धर्मः साधुः सत्यपराक्रमः ।",
                     "Rama is dharma personified: righteous, and of true valour.",
                     "rama dharma righteous ramayana valmiki"),
    "dvaya": V("Dvaya Mantra (Sri Vaishnava tradition)", "श्रीमन्नारायणचरणौ शरणं प्रपद्ये ।\nश्रीमते नारायणाय नमः ॥",
               "I take refuge at the feet of Sriman Narayana, together with Sri. Salutations to Sriman Narayana.",
               "dvaya prapatti sharanagati narayana sri vaishnava refuge acharya"),
    "tiruppavai": V("Andal, Tiruppavai 1 (Divya Prabandham), opening lines", "மார்கழித் திங்கள் மதிநிறைந்த நன்னாளால்\nநீராடப் போதுவீர் போதுமினோ நேரிழையீர்",
                    "In the month of Margazhi, on this bright full-moon day, come, O fair-adorned girls, who wish to bathe (in the vow).",
                    "andal tiruppavai margazhi divya prabandham alvar tamil dhanurmasa"),
    "snana": V("Traditional Snana (Kalasha) Mantra", "गङ्गे च यमुने चैव गोदावरि सरस्वति ।\nनर्मदे सिन्धु कावेरि जलेऽस्मिन् सन्निधिं कुरु ॥",
               "O Ganga, Yamuna, Godavari, Saraswati, Narmada, Sindhu and Kaveri, be present in this water.",
               "rivers ganga kaveri bath snana water puja kalasha"),
    "ahimsa": V("Mahabharata (Anushasana Parva)", "अहिंसा परमो धर्मः",
                "Non-violence is the highest dharma.",
                "ahimsa nonviolence peace compassion"),
}

MAHAVAKYAS = [
    ("प्रज्ञानं ब्रह्म", "Consciousness is Brahman", "Aitareya Upanishad 3.3 (Rigveda)"),
    ("अयमात्मा ब्रह्म", "This Self is Brahman", "Mandukya Upanishad 1.2 (Atharvaveda)"),
    ("तत्त्वमसि", "That thou art", "Chandogya Upanishad 6.8.7 (Samaveda)"),
    ("अहं ब्रह्मास्मि", "I am Brahman", "Brihadaranyaka Upanishad 1.4.10 (Yajurveda)"),
]

GITA_INDEX: dict[tuple[int, int], str] = {}
for _vid, _v in VERSES.items():
    _m = re.match(r"Bhagavad Gita (\d+)\.(\d+)$", _v["ref"])
    if _m:
        GITA_INDEX[(int(_m.group(1)), int(_m.group(2)))] = _vid


def _norm_dev(s: str) -> str:
    return re.sub(r"[^\u0900-\u097F]|[।॥०-९]", "", s)


VERIFIED_NORM = [_norm_dev(v["sa"]) for v in VERSES.values()] + [_norm_dev(m[0]) for m in MAHAVAKYAS]

# ═══════════════════════════════════════════════════════════════════════════
# Bot, command tree, database
# ═══════════════════════════════════════════════════════════════════════════

IST = datetime.timezone(datetime.timedelta(hours=5, minutes=30))

ai_client = OpenAI(api_key=GROQ_API_KEY, base_url="https://api.groq.com/openai/v1", timeout=45.0, max_retries=1)
ai_sem = asyncio.Semaphore(4)           # at most 4 AI calls at once (keeps the free tier happy)
bg_tasks: set = set()                   # keeps references to background tasks (reminders) so they aren't garbage-collected

intents = discord.Intents.default()
intents.members = True
intents.message_content = True


class DharmaTree(app_commands.CommandTree):
    async def interaction_check(self, interaction: discord.Interaction) -> bool:
        if interaction.guild is None:
            await interaction.response.send_message("🙏 These commands only work inside a server.", ephemeral=True)
            return False
        return True

    async def on_error(self, interaction: discord.Interaction, error: app_commands.AppCommandError):
        original = getattr(error, "original", error)
        if isinstance(error, app_commands.MissingPermissions):
            msg = "🚫 You don't have the permission needed for this command."
        elif isinstance(error, app_commands.BotMissingPermissions):
            msg = f"🚫 I'm missing permission(s): `{', '.join(error.missing_permissions)}`."
        elif isinstance(error, app_commands.CommandOnCooldown):
            msg = f"⏳ Easy there! Try again in {error.retry_after:.0f}s."
        elif isinstance(error, app_commands.CheckFailure):
            msg = "🚫 You can't use this command here."
        elif isinstance(original, discord.Forbidden):
            msg = "🚫 Discord says I'm not allowed to do that (check my role position and permissions)."
        else:
            msg = "⚠️ Something went wrong running that command. Please try again."
            print(f"[command error] {type(original).__name__}: {original}")
        try:
            if interaction.response.is_done():
                await interaction.followup.send(msg, ephemeral=True)
            else:
                await interaction.response.send_message(msg, ephemeral=True)
        except discord.HTTPException:
            pass


class DharmaBot(commands.Bot):
    def __init__(self):
        super().__init__(
            command_prefix=commands.when_mentioned,
            intents=intents,
            help_command=None,
            tree_cls=DharmaTree,
            allowed_mentions=discord.AllowedMentions(everyone=False, roles=False, users=True, replied_user=False),
        )
        self.db: Optional[aiosqlite.Connection] = None
        self.start_time = time.time()
        self.xp_cooldowns: dict = {}
        self.chat_cooldowns: dict = {}

    async def setup_hook(self):
        self.db = await aiosqlite.connect(DB_PATH)
        await self.db.executescript(
            """
            CREATE TABLE IF NOT EXISTS levels (
                guild_id INTEGER NOT NULL, user_id INTEGER NOT NULL,
                words INTEGER NOT NULL DEFAULT 0, level INTEGER NOT NULL DEFAULT 0,
                PRIMARY KEY (guild_id, user_id)
            );
            CREATE TABLE IF NOT EXISTS level_roles (
                guild_id INTEGER NOT NULL, level INTEGER NOT NULL, role_id INTEGER NOT NULL,
                PRIMARY KEY (guild_id, level)
            );
            CREATE TABLE IF NOT EXISTS settings (
                guild_id INTEGER NOT NULL, key TEXT NOT NULL, value TEXT,
                PRIMARY KEY (guild_id, key)
            );
            CREATE TABLE IF NOT EXISTS warnings (
                id INTEGER PRIMARY KEY AUTOINCREMENT, guild_id INTEGER NOT NULL, user_id INTEGER NOT NULL,
                moderator_id INTEGER NOT NULL, reason TEXT, created_at TEXT
            );
            CREATE TABLE IF NOT EXISTS banned_words (
                guild_id INTEGER NOT NULL, word TEXT NOT NULL, PRIMARY KEY (guild_id, word)
            );
            CREATE TABLE IF NOT EXISTS reaction_roles (
                guild_id INTEGER NOT NULL, channel_id INTEGER NOT NULL, message_id INTEGER NOT NULL,
                emoji TEXT NOT NULL, role_id INTEGER NOT NULL,
                PRIMARY KEY (guild_id, message_id, emoji)
            );
            """
        )
        await self.db.commit()
        self.add_view(TicketPanelView())
        self.add_view(TicketCloseView())
        try:
            synced = await self.tree.sync()
            print(f"✅ Synced {len(synced)} top-level slash commands (groups count as one each).")
        except Exception as e:
            print(f"⚠️ Slash command sync failed: {e}")

    async def close(self):
        if self.db is not None:
            await self.db.close()
        await super().close()


bot = DharmaBot()

# ═══════════════════════════════════════════════════════════════════════════
# Small helpers
# ═══════════════════════════════════════════════════════════════════════════


def base_embed(title: str = None, description: str = None) -> discord.Embed:
    return discord.Embed(title=title, description=description, color=SAFFRON, timestamp=discord.utils.utcnow())


def info_embed(title: str, body: str, footer: str = "Hari Om 🙏") -> discord.Embed:
    e = base_embed(title=title, description=body[:4000])
    e.set_footer(text=footer)
    return e


def chunk_text(text: str, size: int = 1900) -> list:
    chunks = []
    text = text or ""
    while len(text) > size:
        cut = text.rfind("\n", 0, size)
        if cut < size // 2:
            cut = text.rfind(" ", 0, size)
        if cut < size // 2:
            cut = size
        chunks.append(text[:cut].strip())
        text = text[cut:].strip()
    if text:
        chunks.append(text)
    return chunks or [""]


def verse_embed(vid: str) -> discord.Embed:
    v = VERSES[vid]
    quoted = "\n".join("> " + line for line in v["sa"].split("\n"))
    e = base_embed(title=f"📜 {v['ref']}", description=f"{quoted}\n\n*{v['en']}*")
    e.set_footer(text="Verified verse library · Hari Om 🙏")
    return e


def relevant_verses(query: str, limit: int = 3) -> list:
    words = set(re.findall(r"[a-z]+", query.lower()))
    scored = []
    for vid, v in VERSES.items():
        score = len(words & v["tags"])
        if score:
            scored.append((score, vid))
    scored.sort(reverse=True)
    return [VERSES[vid] for _, vid in scored[:limit]]


def scrub_unverified_verses(text: str) -> str:
    """Safety net: if the AI writes any Sanskrit verse (ends with a double danda) that is NOT in the
    verified library, replace it. Short mantras and ordinary Hindi sentences are left untouched."""
    if "॥" not in text and "||" not in text:
        return text
    has_dev = lambda s: re.search(r"[\u0900-\u097F]", s) is not None
    lines, out, i = text.split("\n"), [], 0
    while i < len(lines):
        if has_dev(lines[i]):
            j = i
            while j < len(lines) and has_dev(lines[j]):
                j += 1
            block = "\n".join(lines[i:j])
            if "॥" in block or "||" in block:
                n = _norm_dev(block)
                if len(n) >= 12 and not any(n in ref for ref in VERIFIED_NORM):
                    block = "*(verse omitted: I only quote verified verses, try /dharma search)*"
            out.append(block)
            i = j
        else:
            out.append(lines[i])
            i += 1
    return "\n".join(out)


def ist_now() -> datetime.datetime:
    return datetime.datetime.now(IST)


# ═══════════════════════════════════════════════════════════════════════════
# Settings (per-server, stored in SQLite)
# ═══════════════════════════════════════════════════════════════════════════


async def get_setting(guild_id: int, key: str, default=None):
    cur = await bot.db.execute("SELECT value FROM settings WHERE guild_id = ? AND key = ?", (guild_id, key))
    row = await cur.fetchone()
    await cur.close()
    return row[0] if row and row[0] is not None else default


async def set_setting(guild_id: int, key: str, value: str):
    await bot.db.execute(
        "INSERT INTO settings (guild_id, key, value) VALUES (?, ?, ?) "
        "ON CONFLICT(guild_id, key) DO UPDATE SET value = excluded.value",
        (guild_id, key, str(value)),
    )
    await bot.db.commit()


async def del_setting(guild_id: int, key: str):
    await bot.db.execute("DELETE FROM settings WHERE guild_id = ? AND key = ?", (guild_id, key))
    await bot.db.commit()


async def channel_setting(guild: discord.Guild, key: str, default_id: Optional[int]):
    raw = await get_setting(guild.id, key)
    cid = int(raw) if raw and raw.isdigit() else default_id
    ch = guild.get_channel(cid) if cid else None
    return ch if ch is not None and hasattr(ch, "send") else None


async def log_event(guild: discord.Guild, embed: discord.Embed):
    ch = await channel_setting(guild, "log_channel", LOG_CHANNEL_ID)
    if ch:
        try:
            await ch.send(embed=embed)
        except (discord.Forbidden, discord.HTTPException):
            pass


def fill_template(text: str, member: discord.Member) -> str:
    return (text.replace("{mention}", member.mention).replace("{name}", member.display_name)
            .replace("{server}", member.guild.name).replace("{count}", str(member.guild.member_count)))


# ═══════════════════════════════════════════════════════════════════════════
# AI layer (Groq, free) with memory, fallback model and verse protection
# ═══════════════════════════════════════════════════════════════════════════

chat_memory: dict = {}


def remember(key: tuple, role: str, content: str):
    h = chat_memory.setdefault(key, [])
    h.append({"role": role, "content": content[:1500]})
    del h[:-12]
    if len(chat_memory) > 400:
        chat_memory.pop(next(iter(chat_memory)))


def ai_ready() -> bool:
    return bool(GROQ_API_KEY) and "PUT_YOUR" not in GROQ_API_KEY


def friendly_ai_error(e: Exception) -> str:
    s = str(e).lower()
    if not ai_ready():
        return "my AI key isn't set up yet. An admin needs to put the free Groq key into main.py 🙏"
    if "401" in s or "api key" in s or "authentication" in s or "invalid_api_key" in s:
        return "my AI key looks invalid. An admin needs to check the Groq key in main.py."
    if "429" in s or "rate" in s:
        return "I'm being rate-limited for a moment, give me a few seconds and try again 🙏"
    if "timed out" in s or "timeout" in s:
        return "the AI server took too long to answer, try again."
    return "something broke on my side while thinking, try again in a bit."


def _chat_sync(messages: list, temperature: float, max_tokens: int) -> str:
    last = None
    for model in (AI_MODEL, AI_FALLBACK_MODEL):
        try:
            r = ai_client.chat.completions.create(model=model, messages=messages, temperature=temperature, max_tokens=max_tokens)
            text = (r.choices[0].message.content or "").strip()
            if text:
                return text
        except Exception as e:  # try the fallback model, then raise the last error
            last = e
    raise last or RuntimeError("empty AI response")


async def _run_ai(messages: list, temperature: float, max_tokens: int) -> str:
    async with ai_sem:
        return await asyncio.get_running_loop().run_in_executor(None, lambda: _chat_sync(messages, temperature, max_tokens))


def build_system(query: str, extra: str = "") -> str:
    verses = relevant_verses(query)
    if verses:
        block = "VERIFIED VERSES (copy exactly if you quote):\n" + "\n\n".join(
            f"[{v['ref']}]\n{v['sa']}\nMeaning: {v['en']}" for v in verses)
    else:
        block = "VERIFIED VERSES: none provided for this message, so do not quote any Sanskrit or Tamil."
    return CHAT_SYSTEM + ("\n\n" + extra if extra else "") + "\n\n" + block


async def ai_reply(user_text: str, history: list, extra: str = "", max_tokens: int = 500, temperature: float = 0.85) -> str:
    messages = [{"role": "system", "content": build_system(user_text, extra)}] + history + [{"role": "user", "content": user_text}]
    return scrub_unverified_verses(await _run_ai(messages, temperature, max_tokens))


async def ai_task(system: str, user: str, max_tokens: int = 700, temperature: float = 0.6, scrub: bool = True) -> str:
    text = await _run_ai([{"role": "system", "content": system}, {"role": "user", "content": user}], temperature, max_tokens)
    return scrub_unverified_verses(text) if scrub else text


# ═══════════════════════════════════════════════════════════════════════════
# Web scraping + deep search (no API keys needed)
# ═══════════════════════════════════════════════════════════════════════════

UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
HEADERS = {
    "User-Agent": UA,
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.9,hi;q=0.6",
}


def normalize_url(url: str) -> str:
    url = url.strip().strip("<>")
    if not re.match(r"^https?://", url, re.I):
        url = "https://" + url
    return url


def is_public_url(url: str) -> bool:
    """SSRF guard: only allow http(s) URLs that resolve to public IP addresses."""
    try:
        parsed = urllib.parse.urlparse(url)
        if parsed.scheme not in ("http", "https") or not parsed.hostname:
            return False
        host = parsed.hostname
        if host.lower() in ("localhost",) or host.lower().endswith((".local", ".internal")):
            return False
        for info in socket.getaddrinfo(host, None):
            ip = ipaddress.ip_address(info[4][0])
            if ip.is_private or ip.is_loopback or ip.is_link_local or ip.is_reserved or ip.is_multicast:
                return False
        return True
    except Exception:
        return False


def fetch_page_sync(url: str, max_chars: int = 6000) -> dict:
    url = normalize_url(url)
    session = requests.Session()
    session.headers.update(HEADERS)
    current, resp = url, None
    for _ in range(6):
        if not is_public_url(current):
            raise ValueError("That address isn't a public website I can open.")
        resp = session.get(current, timeout=(6, 14), allow_redirects=False, stream=True)
        if resp.status_code in (301, 302, 303, 307, 308) and resp.headers.get("Location"):
            current = urllib.parse.urljoin(current, resp.headers["Location"])
            resp.close()
            continue
        break
    else:
        raise ValueError("That link redirects too many times.")
    resp.raise_for_status()
    ctype = resp.headers.get("Content-Type", "").lower()
    if ctype and not any(t in ctype for t in ("html", "text", "xml")):
        raise ValueError(f"That link isn't a web page (it's `{ctype.split(';')[0]}`).")
    raw = resp.raw.read(2_000_000, decode_content=True)
    resp.close()

    soup = BeautifulSoup(raw, "html.parser")
    meta = lambda **kw: (soup.find("meta", attrs=kw) or {}).get("content", "")
    title = (soup.title.get_text(" ", strip=True) if soup.title else "") or meta(property="og:title")
    description = meta(name="description") or meta(property="og:description")
    for tag in soup(["script", "style", "noscript", "nav", "footer", "header", "aside", "form", "svg", "iframe"]):
        tag.decompose()
    headings = []
    for h in soup.find_all(["h1", "h2", "h3"]):
        t = " ".join(h.get_text(" ", strip=True).split())
        if 3 < len(t) < 120 and t not in headings:
            headings.append(t)
    paragraphs, seen = [], set()
    for el in soup.find_all(["p", "li"]):
        t = " ".join(el.get_text(" ", strip=True).split())
        if len(t) >= 40 and t not in seen:
            seen.add(t)
            paragraphs.append(t)
    text = "\n".join(paragraphs)
    return {
        "url": current, "status": resp.status_code, "title": title.strip(), "description": description.strip(),
        "headings": headings[:8], "paragraphs": paragraphs, "text": text[:max_chars], "words": len(text.split()),
    }


def scrape_error_message(e: Exception) -> str:
    if isinstance(e, ValueError):
        return str(e)
    if isinstance(e, requests.exceptions.HTTPError):
        code = e.response.status_code if e.response is not None else "?"
        if code in (401, 403):
            return f"That site refused access (HTTP {code}). It likely blocks bots."
        if code == 404:
            return "That page doesn't exist (HTTP 404)."
        return f"The site answered with an error (HTTP {code})."
    if isinstance(e, requests.exceptions.Timeout):
        return "The site took too long to respond."
    if isinstance(e, requests.exceptions.ConnectionError):
        return "I couldn't connect to that site. Check the link."
    return "I couldn't read that page."


def _unwrap_ddg(href: str) -> str:
    if href.startswith("//"):
        href = "https:" + href
    parsed = urllib.parse.urlparse(href)
    if "duckduckgo.com" in parsed.netloc and parsed.path.startswith("/l/"):
        qs = urllib.parse.parse_qs(parsed.query)
        if "uddg" in qs:
            return qs["uddg"][0]
    return href


def ddg_search_sync(query: str, limit: int = 7) -> list:
    r = requests.post("https://html.duckduckgo.com/html/", data={"q": query}, headers=HEADERS, timeout=(6, 12))
    r.raise_for_status()
    soup = BeautifulSoup(r.text, "html.parser")
    results = []
    for res in soup.select("div.result"):
        if "result--ad" in (res.get("class") or []):
            continue
        a = res.select_one("a.result__a")
        if not a:
            continue
        url = _unwrap_ddg(a.get("href", ""))
        if not url.startswith("http"):
            continue
        sn = res.select_one(".result__snippet")
        results.append({"title": a.get_text(" ", strip=True), "url": url, "snippet": sn.get_text(" ", strip=True) if sn else ""})
        if len(results) >= limit:
            break
    return results


def wiki_search_sync(query: str, limit: int = 5) -> list:
    r = requests.get(
        "https://en.wikipedia.org/w/api.php",
        params={"action": "query", "list": "search", "srsearch": query, "format": "json", "srlimit": limit, "utf8": 1},
        headers=HEADERS, timeout=10,
    )
    r.raise_for_status()
    out = []
    for item in r.json().get("query", {}).get("search", []):
        title = item["title"]
        out.append({
            "title": title,
            "url": "https://en.wikipedia.org/wiki/" + urllib.parse.quote(title.replace(" ", "_")),
            "snippet": BeautifulSoup(item.get("snippet", ""), "html.parser").get_text(),
        })
    return out


def deepsearch_sync(query: str) -> dict:
    results = []
    try:
        results = ddg_search_sync(query)
    except Exception as e:
        print(f"[deepsearch] DuckDuckGo failed: {e}")
    if len(results) < 3:
        try:
            for w in wiki_search_sync(query):
                if all(w["url"] != r["url"] for r in results):
                    results.append(w)
        except Exception as e:
            print(f"[deepsearch] Wikipedia failed: {e}")
    if not results:
        raise RuntimeError("no_results")

    def grab(r: dict) -> dict:
        try:
            r["text"] = fetch_page_sync(r["url"], max_chars=3500)["text"] or r["snippet"]
        except Exception:
            r["text"] = r["snippet"]
        return r

    with ThreadPoolExecutor(max_workers=5) as ex:
        top = list(ex.map(grab, results[:5]))
    sources = [s for s in top if s.get("text")]
    if not sources:
        raise RuntimeError("no_results")
    context = "\n\n".join(f"[{i + 1}] {s['title']} ({s['url']})\n{s['text'][:3000]}" for i, s in enumerate(sources))
    return {"sources": sources, "context": context}


DEEP_SYSTEM = (
    f"You are a careful research assistant for the {PEETHAM} community. Answer the question using ONLY the numbered "
    "sources provided. Put citations like [1] or [2] right after the claims they support. If sources disagree or do not "
    "cover part of the question, say so plainly. Be clear and organised in at most 350 words, with a respectful, "
    "traditional tone. Never invent facts. Do NOT write Sanskrit or Tamil verses unless they appear word-for-word in the "
    "sources; for scripture questions give the meaning instead."
)

# ═══════════════════════════════════════════════════════════════════════════
# Tickets (persistent buttons)
# ═══════════════════════════════════════════════════════════════════════════


def safe_channel_name(name: str) -> str:
    cleaned = re.sub(r"[^a-z0-9\-_]", "", name.lower().replace(" ", "-"))
    return (cleaned or "ticket")[:90]


def ticket_owner_id(channel) -> Optional[int]:
    topic = getattr(channel, "topic", None) or ""
    if topic.startswith("ticket:") and topic[7:].isdigit():
        return int(topic[7:])
    return None


class TicketPanelView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="🎫 Open Verification Ticket", style=discord.ButtonStyle.success, custom_id="open_verification_ticket")
    async def open_ticket(self, interaction: discord.Interaction, button: discord.ui.Button):
        guild, member = interaction.guild, interaction.user
        existing = discord.utils.find(lambda c: ticket_owner_id(c) == member.id, guild.text_channels)
        if existing:
            await interaction.response.send_message(f"You already have an open ticket: {existing.mention}", ephemeral=True)
            return
        overwrites = {
            guild.default_role: discord.PermissionOverwrite(view_channel=False),
            member: discord.PermissionOverwrite(view_channel=True, send_messages=True, read_message_history=True, attach_files=True),
            guild.me: discord.PermissionOverwrite(view_channel=True, send_messages=True, manage_channels=True, read_message_history=True),
        }
        try:
            ticket_channel = await guild.create_text_channel(
                name=safe_channel_name(f"ticket-{member.name}"),
                category=getattr(interaction.channel, "category", None),
                overwrites=overwrites,
                topic=f"ticket:{member.id}",
                reason=f"Verification ticket opened by {member}",
            )
        except discord.Forbidden:
            await interaction.response.send_message("I can't create channels here. Please tell an admin (I need Manage Channels). 🙏", ephemeral=True)
            return
        embed = base_embed(title="🎫 Verification Ticket", description=f"Welcome, {member.mention}! 🙏\n\n{TICKET_QUESTIONS}")
        embed.set_footer(text="Hari Om | A staff member will review your answers shortly")
        await ticket_channel.send(content=member.mention, embed=embed, view=TicketCloseView())
        await interaction.response.send_message(f"Your ticket is ready: {ticket_channel.mention} 🙏", ephemeral=True)
        await log_event(guild, base_embed(title="🎫 Ticket Opened", description=f"{member.mention} opened {ticket_channel.mention}"))


class TicketCloseView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)

    @discord.ui.button(label="🔒 Close Ticket", style=discord.ButtonStyle.danger, custom_id="close_verification_ticket")
    async def close_ticket(self, interaction: discord.Interaction, button: discord.ui.Button):
        channel = interaction.channel
        owner = ticket_owner_id(channel)
        if interaction.user.id != owner and not channel.permissions_for(interaction.user).manage_channels:
            await interaction.response.send_message("Only the ticket owner or staff can close this ticket.", ephemeral=True)
            return
        await interaction.response.send_message("Closing this ticket in 5 seconds... 🙏")
        await log_event(interaction.guild, base_embed(title="🔒 Ticket Closed", description=f"`{channel.name}` closed by {interaction.user.mention}"))
        await asyncio.sleep(5)
        try:
            await channel.delete(reason=f"Ticket closed by {interaction.user}")
        except (discord.Forbidden, discord.NotFound):
            pass


async def post_ticket_panel(guild: discord.Guild) -> Optional[discord.abc.Messageable]:
    ch = await channel_setting(guild, "verify_channel", VERIFY_CHANNEL_ID)
    if ch is None:
        return None
    embed = base_embed(
        title=f"🎫 Verification — {PEETHAM}",
        description="Click the button below to open a private verification ticket. A staff member will assist you shortly. 🙏",
    )
    await ch.send(embed=embed, view=TicketPanelView())
    return ch

# ═══════════════════════════════════════════════════════════════════════════
# Events
# ═══════════════════════════════════════════════════════════════════════════


async def build_welcome_embed(member: discord.Member) -> discord.Embed:
    custom = await get_setting(member.guild.id, "welcome_message")
    body = fill_template(custom, member) if custom else WELCOME_BODY
    embed = base_embed(title=f"Welcome to {PEETHAM}, {member.display_name} 🕉️", description=body)
    embed.set_thumbnail(url=member.display_avatar.url)
    embed.set_footer(text=f"Member #{member.guild.member_count} | Hari Om 🕉️")
    return embed


async def build_goodbye_embed(member: discord.Member) -> discord.Embed:
    custom = await get_setting(member.guild.id, "goodbye_message")
    body = fill_template(custom, member) if custom else GOODBYE_BODY.replace("{mention}", member.mention)
    embed = base_embed(title=f"🕉️ Purna Shanti | Departure from {PEETHAM} 🕉️", description=body)
    embed.set_thumbnail(url=member.display_avatar.url)
    embed.set_footer(text=f"{member.display_name} · {member.guild.member_count} members remain")
    return embed


@bot.event
async def on_ready():
    print(f"🕉️  Logged in as {bot.user} (ID: {bot.user.id}) · {len(bot.guilds)} server(s)")
    await bot.change_presence(activity=discord.Activity(type=discord.ActivityType.listening, name="/help 🙏 Hari Om"))


@bot.event
async def on_raw_reaction_add(payload: discord.RawReactionActionEvent):
    """Persistent reaction roles; works even when the target message is not cached."""
    if payload.guild_id is None or (bot.user and payload.user_id == bot.user.id):
        return
    emoji_key = str(payload.emoji)
    cur = await bot.db.execute(
        "SELECT role_id FROM reaction_roles WHERE guild_id=? AND message_id=? AND emoji=?",
        (payload.guild_id, payload.message_id, emoji_key),
    )
    row = await cur.fetchone()
    await cur.close()
    if not row:
        return
    guild = bot.get_guild(payload.guild_id)
    member = guild.get_member(payload.user_id) if guild else None
    role = guild.get_role(int(row[0])) if guild else None
    if member and role and not member.bot:
        try:
            await member.add_roles(role, reason="Reaction role selected")
        except (discord.Forbidden, discord.HTTPException) as exc:
            print(f"[reaction roles] Could not add role: {exc}")


@bot.event
async def on_raw_reaction_remove(payload: discord.RawReactionActionEvent):
    if payload.guild_id is None or (bot.user and payload.user_id == bot.user.id):
        return
    emoji_key = str(payload.emoji)
    cur = await bot.db.execute(
        "SELECT role_id FROM reaction_roles WHERE guild_id=? AND message_id=? AND emoji=?",
        (payload.guild_id, payload.message_id, emoji_key),
    )
    row = await cur.fetchone()
    await cur.close()
    if not row:
        return
    guild = bot.get_guild(payload.guild_id)
    member = guild.get_member(payload.user_id) if guild else None
    role = guild.get_role(int(row[0])) if guild else None
    if member and role and not member.bot:
        try:
            await member.remove_roles(role, reason="Reaction role deselected")
        except (discord.Forbidden, discord.HTTPException) as exc:
            print(f"[reaction roles] Could not remove role: {exc}")


@bot.event
async def on_member_join(member: discord.Member):
    guild = member.guild
    autorole_id = await get_setting(guild.id, "autorole")
    if autorole_id and autorole_id.isdigit():
        role = guild.get_role(int(autorole_id))
        if role:
            try:
                await member.add_roles(role, reason="Auto-role on join")
            except discord.Forbidden:
                pass

    channel = await channel_setting(guild, "welcome_channel", WELCOME_CHANNEL_ID)
    if channel:
        try:
            await channel.send(content=member.mention, embed=await build_welcome_embed(member),
                               allowed_mentions=discord.AllowedMentions(users=[member]))
        except (discord.Forbidden, discord.HTTPException):
            pass

    await log_event(guild, base_embed(
        title="📥 Member Joined",
        description=f"{member.mention} ({member}) joined.\nAccount created {discord.utils.format_dt(member.created_at, style='R')}",
    ))


@bot.event
async def on_member_remove(member: discord.Member):
    guild = member.guild
    channel = await channel_setting(guild, "goodbye_channel", GOODBYE_CHANNEL_ID)
    if channel:
        try:
            await channel.send(embed=await build_goodbye_embed(member))
        except (discord.Forbidden, discord.HTTPException):
            pass

    await log_event(guild, base_embed(title="📤 Member Left", description=f"{member} (ID: {member.id}) left the server."))


@bot.event
async def on_message_delete(message: discord.Message):
    if message.author.bot or message.guild is None:
        return
    log_ch = await channel_setting(message.guild, "log_channel", LOG_CHANNEL_ID)
    if log_ch is not None and message.channel.id == log_ch.id:
        return
    embed = base_embed(title="🗑️ Message Deleted")
    embed.add_field(name="Author", value=f"{message.author.mention} ({message.author})", inline=False)
    embed.add_field(name="Channel", value=message.channel.mention, inline=False)
    embed.add_field(name="Content", value=(message.content or "*(no text: image, embed or attachment)*")[:1000], inline=False)
    await log_event(message.guild, embed)


# ─── Automod ──────────────────────────────────────────────────────────────


async def banned_match(guild_id: int, content: str) -> bool:
    if not content:
        return False
    lowered = content.lower()
    cur = await bot.db.execute("SELECT word FROM banned_words WHERE guild_id = ?", (guild_id,))
    rows = await cur.fetchall()
    await cur.close()
    return any(w.lower() in lowered for (w,) in rows)


async def run_automod(message: discord.Message) -> bool:
    """Returns True if the message was removed."""
    if (await get_setting(message.guild.id, "automod", "on")) != "on":
        return False
    if not await banned_match(message.guild.id, message.content):
        return False
    try:
        await message.delete()
    except (discord.Forbidden, discord.NotFound):
        return False
    try:
        notice = await message.channel.send(embed=base_embed(
            title="🚫 Message Removed",
            description=f"{message.author.mention}, please keep this sacred space respectful. 🙏"), delete_after=6)
    except discord.HTTPException:
        pass
    await log_event(message.guild, base_embed(
        title="🚫 Automod Deletion",
        description=f"Removed a message from {message.author.mention} in {message.channel.mention} (banned term)."))
    return True


# ─── Human-like AI chat (plain text, no embeds) ───────────────────────────


async def maybe_chat(message: discord.Message) -> bool:
    """Replies like a normal person when the bot is @mentioned, replied to, or in the chat channel."""
    guild = message.guild
    triggered = bot.user in message.mentions
    ref_msg = None
    if message.reference is not None:
        ref_msg = message.reference.resolved
        if ref_msg is None and message.reference.message_id:
            try:
                ref_msg = await message.channel.fetch_message(message.reference.message_id)
            except (discord.NotFound, discord.Forbidden, discord.HTTPException):
                ref_msg = None
        if isinstance(ref_msg, discord.Message) and ref_msg.author == bot.user:
            triggered = True
    chat_ch = await get_setting(guild.id, "chat_channel")
    in_chat_channel = bool(chat_ch) and str(message.channel.id) == chat_ch
    if not (triggered or in_chat_channel):
        return False

    now = time.time()
    if now - bot.chat_cooldowns.get(message.author.id, 0) < 2.5:
        return True
    bot.chat_cooldowns[message.author.id] = now

    text = message.content
    for u in message.mentions:
        text = text.replace(f"<@{u.id}>", "").replace(f"<@!{u.id}>", "")
    text = text.strip()
    if not text:
        text = "hey"
    key = (message.channel.id, message.author.id)
    history = list(chat_memory.get(key, []))
    if isinstance(ref_msg, discord.Message) and ref_msg.author == bot.user and ref_msg.content:
        if not history or history[-1]["content"] != ref_msg.content[:1500]:
            history.append({"role": "assistant", "content": ref_msg.content[:1500]})
    named_text = f"{message.author.display_name}: {text}" if in_chat_channel else text

    none = discord.AllowedMentions.none()
    async with message.channel.typing():
        try:
            answer = await ai_reply(named_text, history, extra="This is a normal casual chat message, keep it short and human.")
        except Exception as e:
            print(f"[chat error] {type(e).__name__}: {e}")
            answer = friendly_ai_error(e)
    remember(key, "user", named_text)
    remember(key, "assistant", answer)
    chunks = chunk_text(answer, 1900)
    try:
        await message.reply(chunks[0], mention_author=False, allowed_mentions=none)
        for extra_chunk in chunks[1:]:
            await message.channel.send(extra_chunk, allowed_mentions=none)
    except discord.HTTPException:
        pass
    return True


# ─── Word-count leveling ──────────────────────────────────────────────────


async def award_words(message: discord.Message):
    guild = message.guild
    key = (guild.id, message.author.id)
    now = time.time()
    if now - bot.xp_cooldowns.get(key, 0) < XP_COOLDOWN_SECONDS:
        return
    count = min(len(message.content.split()), 100)
    if count == 0:
        return
    bot.xp_cooldowns[key] = now

    cur = await bot.db.execute("SELECT words, level FROM levels WHERE guild_id = ? AND user_id = ?", (guild.id, message.author.id))
    row = await cur.fetchone()
    await cur.close()
    words, level = (row[0], row[1]) if row else (0, 0)
    words += count
    new_level = words // WORDS_PER_LEVEL
    await bot.db.execute(
        "INSERT INTO levels (guild_id, user_id, words, level) VALUES (?, ?, ?, ?) "
        "ON CONFLICT(guild_id, user_id) DO UPDATE SET words = excluded.words, level = excluded.level",
        (guild.id, message.author.id, words, max(level, new_level)),
    )
    await bot.db.commit()
    if new_level > level:
        embed = base_embed(
            title="🎉 A New Milestone on the Path!",
            description=f"{message.author.mention} has ascended to **Level {new_level}**! 🪔\n\nKeep walking the path, seeker. 🙏")
        embed.set_thumbnail(url=message.author.display_avatar.url)
        await log_event(guild, embed)
        await give_level_role(message.author, new_level)


async def level_role_id(guild_id: int, level: int) -> Optional[int]:
    cur = await bot.db.execute("SELECT role_id FROM level_roles WHERE guild_id = ? AND level = ?", (guild_id, level))
    row = await cur.fetchone()
    await cur.close()
    return row[0] if row else LEVEL_ROLES.get(level)


async def give_level_role(member: discord.Member, level: int):
    rid = await level_role_id(member.guild.id, level)
    role = member.guild.get_role(rid) if rid else None
    if role:
        try:
            await member.add_roles(role, reason=f"Reached level {level}")
        except discord.Forbidden:
            pass


@bot.event
async def on_message(message: discord.Message):
    if message.author.bot or message.guild is None:
        return
    if await run_automod(message):
        return
    if await maybe_chat(message):
        return
    await award_words(message)

# ═══════════════════════════════════════════════════════════════════════════
# Slash command groups (Discord allows 100 top-level commands, but each group holds up to 25
# subcommands, so these groups together give 180+ real commands)
# ═══════════════════════════════════════════════════════════════════════════


def make_group(name: str, description: str) -> app_commands.Group:
    g = app_commands.Group(name=name, description=description)
    bot.tree.add_command(g)
    return g


def add_static(group: app_commands.Group, name: str, desc: str, title: str, body: str):
    async def cb(interaction: discord.Interaction):
        await interaction.response.send_message(embed=info_embed(title, body))
    cb.__name__ = cb.__qualname__ = f"{group.name}_{name}".replace("-", "_")
    group.add_command(app_commands.Command(name=name, description=desc[:100], callback=cb))


def add_verses(group: app_commands.Group, name: str, desc: str, vids: list):
    async def cb(interaction: discord.Interaction):
        await interaction.response.send_message(embeds=[verse_embed(v) for v in vids])
    cb.__name__ = cb.__qualname__ = f"{group.name}_{name}".replace("-", "_")
    group.add_command(app_commands.Command(name=name, description=desc[:100], callback=cb))


def add_concept(group: app_commands.Group, name: str, desc: str, title: str, body: str, vids: list):
    async def cb(interaction: discord.Interaction):
        e = info_embed(title, body)
        for vid in vids:
            v = VERSES[vid]
            e.add_field(name=f"📜 {v['ref']}", value=f"{v['sa']}\n*{v['en']}*"[:1020], inline=False)
        await interaction.response.send_message(embed=e)
    cb.__name__ = cb.__qualname__ = f"{group.name}_{name}".replace("-", "_")
    group.add_command(app_commands.Command(name=name, description=desc[:100], callback=cb))


# ─── /dharma ──────────────────────────────────────────────────────────────

dharma = make_group("dharma", "Verified verses, mantras and core dharma concepts")


@dharma.command(name="verse", description="A random verified verse or mantra")
async def dharma_verse(interaction: discord.Interaction):
    await interaction.response.send_message(embed=verse_embed(random.choice(list(VERSES))))


@dharma.command(name="verseofday", description="Today's verse (same for everyone, changes daily at midnight IST)")
async def dharma_verseofday(interaction: discord.Interaction):
    ids = sorted(VERSES)
    vid = ids[ist_now().date().toordinal() % len(ids)]
    e = verse_embed(vid)
    e.title = f"🌅 Verse of the Day — {VERSES[vid]['ref']}"
    await interaction.response.send_message(embed=e)


@dharma.command(name="gita", description="Look up a Bhagavad Gita verse from the verified library")
@app_commands.describe(chapter="Chapter number (1-18)", verse="Verse number")
async def dharma_gita(interaction: discord.Interaction, chapter: app_commands.Range[int, 1, 18], verse: app_commands.Range[int, 1, 78]):
    vid = GITA_INDEX.get((chapter, verse))
    if vid:
        await interaction.response.send_message(embed=verse_embed(vid))
        return
    have = ", ".join(f"{c}.{v}" for (c, v) in sorted(GITA_INDEX) if c == chapter)
    msg = (f"🙏 Gita {chapter}.{verse} isn't in my verified library yet. I only show verses I can guarantee are exact, "
           f"so I never guess.\n" + (f"Verified verses I have from chapter {chapter}: **{have}**.\n" if have else "")
           + "For the full text, the IIT Kanpur Gita Supersite (gitasupersite.iitk.ac.in) is a reliable source.")
    await interaction.response.send_message(msg, ephemeral=True)


@dharma.command(name="search", description="Search verified verses by topic (karma, bhakti, anger, guru, peace...)")
@app_commands.describe(topic="A keyword such as karma, bhakti, anger, guru, peace, truth")
async def dharma_search(interaction: discord.Interaction, topic: str):
    q = topic.lower().strip()
    words = set(re.findall(r"[a-z]+", q))
    hits = []
    for vid, v in VERSES.items():
        score = len(words & v["tags"]) * 3 + (2 if q in v["ref"].lower() else 0) + (1 if q in v["en"].lower() else 0)
        if score:
            hits.append((score, vid))
    hits.sort(reverse=True)
    if not hits:
        await interaction.response.send_message(
            "🙏 No verified verse matched that. Try topics like **karma, bhakti, anger, guru, peace, truth, dharma, atma, varna, knowledge**.",
            ephemeral=True)
        return
    await interaction.response.send_message(embeds=[verse_embed(vid) for _, vid in hits[:3]])


for _name, _desc, _vids in [
    ("gayatri", "The Gayatri Mantra (Rigveda 3.62.10)", ["gayatri"]),
    ("mrityunjaya", "The Maha Mrityunjaya Mantra (Rigveda 7.59.12)", ["mrityunjaya"]),
    ("shanti", "Shanti mantras: prayers for peace and harmony", ["saha", "sarve"]),
    ("asatoma", "Asato Ma Sad Gamaya (Brihadaranyaka Upanishad)", ["asatoma"]),
    ("purnam", "Purnamadah, the Shanti mantra of wholeness", ["purnam"]),
    ("guru", "The Guru shloka: salutations to the teacher", ["guru"]),
    ("ganesha", "Ganesha shlokas: to remove obstacles", ["vakratunda", "shuklambara"]),
    ("hanuman", "Hanuman shloka (Manojavam Marutatulyavegam)", ["manojavam"]),
    ("vishnu", "Vishnu dhyana shloka (Shantakaram)", ["shantakaram"]),
    ("shiva", "Shiva dhyana shloka (Karpuragauram)", ["karpura"]),
    ("durga", "Devi stuti (Sarvamangala Mangalye)", ["sarvamangala"]),
    ("saraswati", "Saraswati pranama: before study or exams", ["saraswati"]),
    ("rama", "Sri Rama: the Rama nama shloka and a Ramayana verse", ["rama-nama", "ramo-dharma"]),
    ("krishna", "Krishna pranama (Krishnaya Vasudevaya)", ["krishna"]),
    ("dvaya", "The Dvaya Mantra of the Sri Vaishnava tradition", ["dvaya"]),
    ("tiruppavai", "Andal's Tiruppavai: opening lines", ["tiruppavai"]),
]:
    add_verses(dharma, _name, _desc, _vids)


@dharma.command(name="mahavakyas", description="The four Mahavakyas (great Upanishadic statements)")
async def dharma_mahavakyas(interaction: discord.Interaction):
    e = info_embed("🕉️ The Four Mahavakyas", "Great statements drawn from the four Vedas:")
    for sa, en, ref in MAHAVAKYAS:
        e.add_field(name=sa, value=f"*{en}*\n{ref}", inline=False)
    await interaction.response.send_message(embed=e)


VARNA_BODY = (
    "In the traditional (orthodox Dharmashastra) understanding, **varna is determined by janma (birth)**, and each varna "
    "has its own **svadharma** within varnashrama-dharma. The classical listing of duties (Manusmriti 1.88-91):\n\n"
    "**Brahmana:** study and teaching of the Veda, performing and officiating yajna, giving and receiving dana.\n"
    "**Kshatriya:** protecting the people, giving, performing yajna, study.\n"
    "**Vaishya:** tending cattle, agriculture, trade, giving, performing yajna, study.\n"
    "**Shudra:** service (seva) to the other varnas.\n\n"
    "*Interpretation note:* Gita 4.13 and 18.41-42 are the key passages. The orthodox reading ties these gunas and karmas to "
    "one's birth-given nature, while some other teachers and modern traditions read guna-karma as the basis instead. The "
    "traditional view is presented here as taught, and the difference of interpretation is acknowledged openly.\n\n"
    "The same Gita also teaches *samadarshitva* (5.18): the wise see all beings with equal vision. A difference of duty "
    "never means a difference in the dignity of any person before Bhagavan."
)
add_concept(dharma, "varna", "Varna in the traditional (janma-based) understanding, with verses",
            "🕉️ Varna — The Traditional View", VARNA_BODY, ["bg-4-13", "bg-18-41", "bg-18-42", "bg-5-18"])
add_concept(dharma, "karma", "The law of karma and nishkama karma, with verses",
            "🕉️ Karma",
            "Karma means *action*. By the law of karma every deed (kriya) bears a fruit (phala), traditionally understood in three "
            "streams: **sanchita** (accumulated), **prarabdha** (ripened and being experienced now) and **agami** (being created by "
            "present actions).\n\nThe Gita's teaching is *nishkama karma*: do your duty fully, free from craving for the result, "
            "offering the fruit to Bhagavan.", ["bg-2-47", "bg-2-48"])
add_concept(dharma, "moksha", "Moksha: liberation, as understood by different schools",
            "🕉️ Moksha",
            "**Moksha** is liberation from samsara, the cycle of birth and death, and the fourth purushartha. The schools describe "
            "it differently:\n\n**Advaita:** realisation of the Self's identity with Brahman.\n**Vishishtadvaita:** eternal, loving "
            "service (kainkarya) to Sriman Narayana in Sri Vaikuntha, reached through bhakti or prapatti and Bhagavan's grace.\n"
            "**Dvaita:** eternal bliss in the presence of Vishnu, while remaining distinct from Him.", ["bg-18-66", "asatoma"])
add_concept(dharma, "atma", "The Atma (the eternal Self) and how schools understand it",
            "🕉️ Atma — The Eternal Self",
            "The **Atma** is the eternal, unborn and unchanging Self, distinct from body and mind. The body is born, changes and "
            "dies; the Atma does not.\n\nSchools differ on its relation to the Supreme: **Advaita** teaches non-difference from "
            "Brahman; **Vishishtadvaita** teaches an eternal, inseparable yet distinct relationship (the jiva is a 'body' of "
            "Ishvara and His servant); **Dvaita** teaches eternal distinction.", ["bg-2-20", "bg-2-22"])

# ─── /shastra ─────────────────────────────────────────────────────────────

shastra = make_group("shastra", "Scriptures, saints, temples, and sacred lists")

SHASTRA_INFO = [
    ("vedas", "The four Vedas and their four layers", "📚 The Four Vedas",
     "**Rigveda:** hymns of praise (suktas) to the devas.\n**Yajurveda:** sacrificial formulas (Shukla and Krishna branches).\n"
     "**Samaveda:** melodies and chants, largely drawn from the Rigveda.\n**Atharvaveda:** hymns, prayers and mantras for daily life.\n\n"
     "Each Veda has four layers: **Samhita** (mantras), **Brahmana** (ritual explanation), **Aranyaka** (forest texts) and "
     "**Upanishad** (philosophy)."),
    ("upanishads", "The ten principal Upanishads", "📚 The Principal Upanishads",
     "**Isha, Kena, Katha, Prashna, Mundaka, Mandukya, Taittiriya, Aitareya, Chandogya, Brihadaranyaka**: the *Dashopanishad*, "
     "commented upon by Adi Shankaracharya.\n\nThe Muktika canon counts 108 Upanishads in all."),
    ("puranas", "The 18 Mahapuranas", "📚 The Eighteen Mahapuranas",
     "Brahma, Padma, Vishnu, Shiva, Bhagavata, Narada, Markandeya, Agni, Bhavishya, Brahmavaivarta, Linga, Varaha, Skanda, "
     "Vamana, Kurma, Matsya, Garuda and Brahmanda.\n\nTraditionally each Purana has five marks (pancha-lakshana): creation, "
     "re-creation, genealogies, Manvantaras and dynastic histories."),
    ("itihasa", "Ramayana and Mahabharata", "📚 The Itihasas",
     "Valmiki's **Ramayana** (traditionally 24,000 verses) and Vyasa's **Mahabharata** (the longest epic, about 100,000 verses). "
     "The Mahabharata contains the **Bhagavad Gita** (Bhishma Parva) and the **Vishnu Sahasranama** (Anushasana Parva)."),
    ("darshanas", "The six darshanas and the three Vedanta schools", "📚 The Darshanas",
     "**Nyaya** (Gautama), **Vaisheshika** (Kanada), **Samkhya** (Kapila), **Yoga** (Patanjali), **Purva Mimamsa** (Jaimini), "
     "**Uttara Mimamsa / Vedanta** (Badarayana).\n\nMajor Vedanta schools: **Advaita** (Adi Shankara), **Vishishtadvaita** "
     "(Sri Ramanuja), **Dvaita** (Sri Madhva)."),
    ("vedangas", "The six limbs of the Veda", "📚 The Six Vedangas",
     "**Shiksha** (phonetics), **Chandas** (metre), **Vyakarana** (grammar), **Nirukta** (etymology), **Jyotisha** (astronomy "
     "and timekeeping), **Kalpa** (ritual procedure)."),
    ("upavedas", "The four Upavedas", "📚 The Upavedas",
     "**Ayurveda** (medicine), **Dhanurveda** (archery and warfare), **Gandharvaveda** (music and arts), **Sthapatyaveda** (architecture)."),
    ("agamas", "Agamas: the scriptures of temple worship", "📚 The Agamas",
     "The Agamas govern temple building, deity installation and worship. Main streams: **Shaiva Agamas** (28 principal), "
     "**Vaishnava Agamas** (Pancharatra and Vaikhanasa) and **Shakta Agamas / Tantras**.\n\nIn Dakshina Bharata, Tirumala "
     "follows the Vaikhanasa Agama and Srirangam the Pancharatra Agama."),
    ("alvars", "The 12 Alvars and the Divya Prabandham", "📚 The Twelve Alvars",
     "**Poigai, Bhutam, Pey, Tirumazhisai, Nammalvar, Madhurakavi, Kulasekhara, Periyalvar, Andal, Tondaradippodi, Tiruppan, "
     "Tirumangai.**\n\nTheir hymns, compiled by Nathamuni, form the **Nalayira Divya Prabandham** (4,000 verses), which includes "
     "Andal's *Tiruppavai* and Nammalvar's *Tiruvaymozhi*."),
    ("nayanmars", "Tevaram, Tiruvacakam and the Shaiva saints", "📚 The Nayanmars",
     "The **63 Nayanmars** are the Shaiva saints of Tamil Nadu. The **Tevaram** hymns of **Sambandar, Appar (Tirunavukkarasar) and "
     "Sundarar** form Tirumurais 1-7; **Manikkavacakar's Tiruvacakam** is the eighth; **Tirumular's Tirumantiram** the tenth."),
    ("acharyas", "Key Acharyas of the Sri Vaishnava and Vedanta traditions", "🪔 Acharyas",
     "**Sri Vaishnava line:** Nathamuni → Uyyakkondar → Manakkal Nambi → Alavandar (Yamunacharya) → Periya Nambi → "
     "**Sri Ramanujacharya** → later acharyas such as Pillai Lokacharya, Vedanta Desika and Manavala Mamunigal.\n\n"
     "**Other great Acharyas:** Adi Shankaracharya (Advaita) and Madhvacharya (Dvaita)."),
    ("divya-desams", "The 108 Divya Desams and famous examples", "🛕 The Divya Desams",
     "108 sacred Vishnu temples praised by the Alvars (106 on earth, plus Tiruppar-kadal and Sri Vaikuntha).\n\n**Famous examples:** "
     "Srirangam (Sri Ranganatha), Tirumala (Sri Venkateshwara), Kanchipuram (Varadaraja), Tiruvallikeni (Parthasarathy), "
     "Tirukkurungudi, Alagar Koil, Tirukkoshtiyur, Tiruvahindrapuram, Badrinath, Dwarka, Mathura and Ayodhya."),
    ("jyotirlingas", "The 12 Jyotirlinga temples of Shiva", "🛕 The Twelve Jyotirlingas",
     "Somnath (Gujarat), Mallikarjuna (Srisailam), Mahakaleshwar (Ujjain), Omkareshwar (MP), Kedarnath (Uttarakhand), "
     "Bhimashankar (Maharashtra), Kashi Vishwanath (Varanasi), Trimbakeshwar (Maharashtra), Vaidyanath (Deoghar), "
     "Nageshwar (Gujarat), Rameshwaram (Tamil Nadu) and Grishneshwar (Maharashtra).\n\n*The exact sites of Vaidyanath and Nageshwar are traditionally debated.*"),
    ("char-dham", "The four great pilgrimage dhams", "🛕 Char Dham",
     "**Badrinath** (north, Uttarakhand), **Dwarka** (west, Gujarat), **Puri** (east, Odisha) and **Rameswaram** (south, Tamil Nadu).\n\n"
     "The *Chota Char Dham* of the Himalayas: Yamunotri, Gangotri, Kedarnath and Badrinath."),
    ("pancha-bhuta", "Pancha Bhuta Sthalams of Shiva", "🛕 Pancha Bhuta Sthalams",
     "**Earth:** Ekambareswarar, Kanchipuram\n**Water:** Jambukeswarar, Tiruvanaikka\n**Fire:** Arunachaleswarar, Tiruvannamalai\n"
     "**Air:** Srikalahasti (Andhra Pradesh)\n**Space:** Thillai Nataraja, Chidambaram"),
    ("navagraha", "The nine grahas", "🪐 The Navagraha",
     "**Surya** (Sun), **Chandra** (Moon), **Mangala** (Mars), **Budha** (Mercury), **Guru / Brihaspati** (Jupiter), "
     "**Shukra** (Venus), **Shani** (Saturn), **Rahu** and **Ketu** (lunar nodes)."),
    ("dashavatara", "The ten avataras of Vishnu", "🐚 The Dashavatara",
     "**Matsya** (fish), **Kurma** (tortoise), **Varaha** (boar), **Narasimha** (man-lion), **Vamana** (dwarf), **Parashurama**, "
     "**Rama**, **Krishna**, **Buddha** and **Kalki** (yet to come).\n\n*Some traditions list Balarama in place of Buddha.*"),
    ("tithis", "The lunar days of the Hindu month", "🌙 The Tithis",
     "Pratipada, Dwitiya, Tritiya, Chaturthi, Panchami, Shashthi, Saptami, Ashtami, Navami, Dashami, Ekadashi, Dwadashi, "
     "Trayodashi, Chaturdashi, and **Purnima** (full moon) or **Amavasya** (new moon).\n\nThe month has two pakshas: "
     "**Shukla** (waxing) and **Krishna** (waning)."),
    ("nakshatras", "The 27 nakshatras", "⭐ The 27 Nakshatras",
     "Ashwini, Bharani, Krittika, Rohini, Mrigashira, Ardra, Punarvasu, Pushya, Ashlesha, Magha, Purva Phalguni, Uttara Phalguni, "
     "Hasta, Chitra, Swati, Vishakha, Anuradha, Jyeshtha, Mula, Purva Ashadha, Uttara Ashadha, Shravana, Dhanishta, "
     "Shatabhisha, Purva Bhadrapada, Uttara Bhadrapada, Revati."),
    ("rashis", "The 12 rashis (zodiac signs)", "♈ The 12 Rashis",
     "Mesha, Vrishabha, Mithuna, Karka, Simha, Kanya, Tula, Vrishchika, Dhanus, Makara, Kumbha, Meena."),
    ("masas", "Hindu lunar and Tamil solar months", "🗓️ The Months",
     "**Lunar (Sanskrit):** Chaitra, Vaishakha, Jyeshtha, Ashadha, Shravana, Bhadrapada, Ashvina, Kartika, Margashirsha, Pausha, "
     "Magha, Phalguna.\n\n**Tamil solar:** Chithirai, Vaikasi, Aani, Aadi, Aavani, Purattasi, Aippasi, Karthigai, Margazhi, Thai, "
     "Maasi, Panguni."),
    ("rtus", "The six seasons", "🌿 The Six Rtus",
     "**Vasanta** (spring), **Grishma** (summer), **Varsha** (monsoon), **Sharad** (autumn), **Hemanta** (pre-winter), **Shishira** (winter)."),
    ("gotras", "Gotra and the Saptarishis", "🌳 Gotra",
     "A **gotra** is a lineage traced through the male line to a Rishi. The seven rishis most commonly listed are **Atri, "
     "Bharadvaja, Gautama, Jamadagni, Kashyapa, Vasishtha and Vishvamitra** (some lists add Agastya)."),
    ("sapta-puri", "The seven moksha-giving cities", "🛕 The Sapta Puri",
     "**Ayodhya, Mathura, Maya (Haridwar), Kashi, Kanchi, Avantika (Ujjain) and Dvaraka.**"),
    ("chiranjeevi", "The seven chiranjeevis (immortals)", "🪔 The Chiranjeevis",
     "**Ashwatthama, Bali, Vyasa, Hanuman, Vibhishana, Kripacharya and Parashurama** (some traditions add Markandeya)."),
]
for _n, _d, _t, _b in SHASTRA_INFO:
    add_static(shastra, _n, _d, _t, _b)

# ─── /learn ───────────────────────────────────────────────────────────────

learn = make_group("learn", "Learn Sanskrit basics, concepts and traditions")

LEARN_INFO = [
    ("numbers", "Sanskrit numbers 1 to 10", "🔢 Sanskrit Numbers",
     "1 · **eka** (एक)\n2 · **dvi** (द्वि)\n3 · **tri** (त्रि)\n4 · **chatur** (चतुर्)\n5 · **pancha** (पञ्च)\n6 · **shat** (षट्)\n"
     "7 · **sapta** (सप्त)\n8 · **ashta** (अष्ट)\n9 · **nava** (नव)\n10 · **dasha** (दश)"),
    ("devanagari", "The Devanagari vowels and consonants", "🔤 Devanagari",
     "**Vowels:** अ आ इ ई उ ऊ ऋ ॠ ऌ ए ऐ ओ औ अं अः\n\n**Consonants:**\nक ख ग घ ङ\nच छ ज झ ञ\nट ठ ड ढ ण\nत थ द ध न\nप फ ब भ म\n"
     "य र ल व\nश ष स ह"),
    ("greetings", "Traditional greetings and their traditions", "🙏 Traditional Greetings",
     "**Namaste:** the common greeting across Bharata.\n**Namo Narayanaya:** a Sri Vaishnava greeting.\n**Om Namah Shivaya:** a Shaiva greeting.\n"
     "**Hari Om:** a greeting and a prayer.\n**Jai Shri Ram / Jai Shri Krishna / Radhe Radhe:** devotional greetings.\n"
     "**Vanakkam:** Tamil.\n**Namaskaram:** Telugu, Kannada and Malayalam regions.\n**Nomoshkar:** Bengali.\n\n"
     "*Adiyen* (Tamil, 'this servant') is the humble first-person word used by Sri Vaishnavas."),
    ("tilak", "Common tilak and pundra traditions", "🔱 Tilak",
     "**Urdhva Pundra** (a vertical mark, often U or Y shaped, with Sri Churnam) is worn by Vaishnavas.\n"
     "**Tripundra** (three horizontal lines of vibhuti) is worn by Shaivas.\nShakta traditions commonly use a red kumkum mark."),
    ("panchayajna", "The five daily duties of a householder", "🪔 Pancha Mahayajna",
     "**Brahma yajna:** study (svadhyaya)\n**Deva yajna:** worship and homa\n**Pitr yajna:** tarpana and honouring ancestors\n"
     "**Bhuta yajna:** feeding animals and creatures\n**Manushya yajna:** hospitality to guests"),
    ("ashtanga-yoga", "Patanjali's eight limbs of yoga", "🧘 Ashtanga Yoga",
     "**Yama** (restraints), **Niyama** (observances), **Asana** (posture), **Pranayama** (breath), **Pratyahara** (withdrawal of senses), "
     "**Dharana** (concentration), **Dhyana** (meditation), **Samadhi** (absorption). (Yoga Sutra 2.29)"),
    ("yamas-niyamas", "The yamas and niyamas", "🧘 Yamas and Niyamas",
     "**Yamas:** ahimsa, satya, asteya, brahmacharya, aparigraha.\n**Niyamas:** shaucha, santosha, tapas, svadhyaya, ishvara-pranidhana."),
    ("trimurti", "Brahma, Vishnu, Shiva", "🔱 The Trimurti",
     "**Brahma** (creation), **Vishnu** (preservation), **Shiva** (dissolution).\n\nSampradayas hold different views of the Supreme: "
     "Vaishnavas worship Narayana, Shaivas worship Shiva, and Shaktas worship Devi."),
    ("tridevi", "Saraswati, Lakshmi, Parvati", "🌸 The Tridevi",
     "**Saraswati** (knowledge), **Lakshmi** (fortune and prosperity), **Parvati / Durga** (strength)."),
    ("gunas", "Sattva, rajas and tamas", "⚖️ The Three Gunas",
     "**Sattva:** clarity, harmony, knowledge.\n**Rajas:** passion, activity, restlessness.\n**Tamas:** inertia, dullness, ignorance.\n\nDescribed in Gita chapters 14 and 17-18."),
    ("purusharthas", "The four aims of life", "🎯 The Purusharthas",
     "**Dharma** (righteousness), **Artha** (livelihood), **Kama** (legitimate desire), **Moksha** (liberation)."),
    ("ashramas", "The four stages of life", "🛤️ The Four Ashramas",
     "**Brahmacharya** (student), **Grihastha** (householder), **Vanaprastha** (forest-dweller), **Sannyasa** (renunciate)."),
    ("yugas", "The four yugas and their durations", "⏳ The Four Yugas",
     "**Satya (Krita):** 1,728,000 years\n**Treta:** 1,296,000 years\n**Dvapara:** 864,000 years\n**Kali:** 432,000 years\n\n"
     "Together they make one Mahayuga of 4,320,000 years. Kali Yuga begins in 3102 BCE by traditional reckoning."),
    ("samskaras", "The sixteen samskaras", "🪔 The Sixteen Samskaras",
     "Garbhadhana, Pumsavana, Simantonnayana, Jatakarma, Namakarana, Nishkramana, Annaprashana, Chudakarana, Karnavedha, "
     "Vidyarambha, Upanayana, Vedarambha, Keshanta, Samavartana, Vivaha and Antyeshti.\n\n*Lists vary slightly between Grihya Sutras.*"),
    ("ekadashi", "What Ekadashi is and how it is observed", "🌙 Ekadashi",
     "Ekadashi is the 11th tithi of each paksha, about 24 a year, observed with fasting and vigil dedicated to Vishnu. The fast is "
     "broken (parana) on Dwadashi within the proper time.\n\n**Vaikuntha Ekadashi** in Margazhi (Dhanurmasa) is especially "
     "celebrated in Sri Vaishnava temples such as Srirangam."),
    ("vaara", "The seven days and their grahas", "📅 The Seven Days",
     "**Ravivasara** (Sun) · **Somavasara** (Moon) · **Mangalavasara** (Mars) · **Budhavasara** (Mercury) · **Guruvasara** (Jupiter) · "
     "**Shukravasara** (Venus) · **Shanivasara** (Saturn)"),
    ("sapta-nadi", "The seven sacred rivers", "🌊 The Sapta Nadi",
     "**Ganga, Yamuna, Godavari, Sarasvati, Narmada, Sindhu and Kaveri.** (See `/dharma search rivers` for the invocation.)"),
    ("pancha-kosha", "The five sheaths of the self", "🧅 The Pancha Kosha",
     "**Annamaya** (physical), **Pranamaya** (vital), **Manomaya** (mental), **Vijnanamaya** (intellectual), **Anandamaya** (bliss). "
     "(Taittiriya Upanishad, Brahmananda Valli)"),
    ("four-yogas", "Karma, Jnana, Bhakti and Raja yoga", "🧘 The Four Yogas",
     "**Karma yoga:** selfless action\n**Jnana yoga:** knowledge\n**Bhakti yoga:** devotion\n**Raja yoga:** meditation\n\nThe Gita teaches the first three in depth."),
    ("navavidha-bhakti", "The nine modes of devotion", "🌺 Navavidha Bhakti",
     "**Shravana** (hearing), **Kirtana** (singing), **Smarana** (remembering), **Pada-sevana** (service at the feet), **Archana** (worship), "
     "**Vandana** (salutation), **Dasya** (servitude), **Sakhya** (friendship), **Atma-nivedana** (total surrender). (Bhagavata Purana 7.5.23)"),
    ("shad-ripu", "The six inner enemies", "⚔️ The Shad Ripu",
     "**Kama** (lust), **Krodha** (anger), **Lobha** (greed), **Moha** (delusion), **Mada** (pride), **Matsarya** (envy)."),
    ("tattvatraya", "The three realities in Vishishtadvaita", "🪔 Tattvatraya",
     "Sri Ramanuja's Vishishtadvaita teaches three eternal realities: **Chit** (the individual souls), **Achit** (matter) and "
     "**Ishvara** (Sriman Narayana, with Sri). Chit and Achit are the body of Ishvara, who is their inner self."),
    ("bhagavan-forms", "The five forms of Bhagavan (Pancharatra)", "🪔 The Five Forms",
     "**Para** (in Sri Vaikuntha), **Vyuha** (Vasudeva, Sankarshana, Pradyumna, Aniruddha), **Vibhava** (the avataras), "
     "**Antaryami** (the indweller in all hearts) and **Archa** (the deity in temples and homes)."),
    ("dasha-mahavidya", "The ten Mahavidyas", "🌺 The Dasha Mahavidya",
     "Kali, Tara, Tripura Sundari (Shodashi), Bhuvaneshvari, Bhairavi, Chhinnamasta, Dhumavati, Bagalamukhi, Matangi and Kamala."),
    ("ashta-dikpala", "The eight guardians of the directions", "🧭 The Ashta Dikpalas",
     "**Indra** (East), **Agni** (South-East), **Yama** (South), **Nirriti** (South-West), **Varuna** (West), **Vayu** (North-West), "
     "**Kubera** (North), **Ishana** (North-East)."),
]
for _n, _d, _t, _b in LEARN_INFO:
    add_static(learn, _n, _d, _t, _b)

# ─── /ai ──────────────────────────────────────────────────────────────────

ai = make_group("ai", "Chat with the AI guide, deep web search and web tools")


async def send_plain(interaction: discord.Interaction, text: str, ephemeral: bool = False):
    for chunk in chunk_text(text, 1900):
        await interaction.followup.send(chunk, ephemeral=ephemeral, allowed_mentions=discord.AllowedMentions.none())


async def answer_question(interaction: discord.Interaction, question: str, ephemeral: bool):
    await interaction.response.defer(thinking=True, ephemeral=ephemeral)
    key = (interaction.channel_id, interaction.user.id)
    try:
        answer = await ai_reply(question, list(chat_memory.get(key, [])), extra=ASK_EXTRA, max_tokens=700)
        remember(key, "user", question)
        remember(key, "assistant", answer)
    except Exception as e:
        print(f"[ask error] {type(e).__name__}: {e}")
        answer = friendly_ai_error(e)
    await send_plain(interaction, answer, ephemeral)


@ai.command(name="ask", description="Ask Sanatana Margadarshak anything (replies like a normal chat)")
@app_commands.describe(question="Your question or message")
@app_commands.checks.cooldown(1, 6.0)
async def ai_ask(interaction: discord.Interaction, question: str):
    await answer_question(interaction, question, False)


@ai.command(name="askprivate", description="Ask a question and get the answer only you can see")
@app_commands.describe(question="Your question or message")
@app_commands.checks.cooldown(1, 6.0)
async def ai_askprivate(interaction: discord.Interaction, question: str):
    await answer_question(interaction, question, True)


@ai.command(name="deepsearch", description="Search the web, read the top pages and give a sourced answer")
@app_commands.describe(query="What do you want to research?")
@app_commands.checks.cooldown(1, 20.0)
async def ai_deepsearch(interaction: discord.Interaction, query: str):
    await interaction.response.defer(thinking=True)
    try:
        data = await asyncio.get_running_loop().run_in_executor(None, deepsearch_sync, query)
    except RuntimeError:
        await interaction.followup.send(embed=info_embed("🔎 Deep Search", "I couldn't find usable results for that. Try different keywords."))
        return
    except Exception as e:
        print(f"[deepsearch error] {type(e).__name__}: {e}")
        await interaction.followup.send(embed=info_embed("🔎 Deep Search", "The web search failed just now. Please try again in a moment."))
        return
    try:
        answer = await ai_task(DEEP_SYSTEM, f"Question: {query}\n\nSOURCES:\n{data['context']}", max_tokens=900, temperature=0.3)
    except Exception as e:
        print(f"[deepsearch ai error] {type(e).__name__}: {e}")
        answer = f"*(AI summary unavailable: {friendly_ai_error(e)})*\n\nHere are the top results I found:"
    embed = base_embed(title=f"🔎 Deep Search: {query[:200]}", description=answer[:4000])
    sources = "\n".join(f"**[{i + 1}]** [{s['title'][:70] or s['url'][:70]}]({s['url']})" for i, s in enumerate(data["sources"]))
    embed.add_field(name="Sources", value=sources[:1020] or "—", inline=False)
    embed.set_footer(text="Answer is based only on the pages above · Hari Om 🙏")
    await interaction.followup.send(embed=embed)


@ai.command(name="scrape", description="Fetch a web page: title, description, headings and a text preview")
@app_commands.describe(url="The web page address")
@app_commands.checks.cooldown(1, 8.0)
async def ai_scrape(interaction: discord.Interaction, url: str):
    await interaction.response.defer(thinking=True)
    try:
        page = await asyncio.get_running_loop().run_in_executor(None, fetch_page_sync, url)
    except Exception as e:
        await interaction.followup.send(embed=info_embed("⚠️ Couldn't Read That Page", scrape_error_message(e)))
        return
    embed = base_embed(title="🔎 Web Page Retrieved")
    embed.add_field(name="🔗 URL", value=page["url"][:1020], inline=False)
    embed.add_field(name="📜 Title", value=(page["title"] or "No title found")[:1020], inline=False)
    if page["description"]:
        embed.add_field(name="🧾 Description", value=page["description"][:500], inline=False)
    if page["headings"]:
        embed.add_field(name="🔖 Headings", value="\n".join(f"• {h[:80]}" for h in page["headings"][:5])[:1020], inline=False)
    preview = "\n\n".join(page["paragraphs"][:3])
    if len(preview) > 900:
        preview = preview[:900].rsplit(" ", 1)[0] + "…"
    embed.add_field(name="📝 Preview", value=preview or "No readable paragraph text found on this page.", inline=False)
    embed.set_footer(text=f"{page['words']} words of text · HTTP {page['status']} · Hari Om 🙏")
    await interaction.followup.send(embed=embed)


@ai.command(name="summarize", description="Read a web page and summarize it")
@app_commands.describe(url="The web page address")
@app_commands.checks.cooldown(1, 10.0)
async def ai_summarize(interaction: discord.Interaction, url: str):
    await interaction.response.defer(thinking=True)
    try:
        page = await asyncio.get_running_loop().run_in_executor(None, fetch_page_sync, url, 9000)
    except Exception as e:
        await interaction.followup.send(embed=info_embed("⚠️ Couldn't Read That Page", scrape_error_message(e)))
        return
    if len(page["text"]) < 200:
        await interaction.followup.send(embed=info_embed("⚠️ Not Enough Text", "That page doesn't have enough readable text to summarize (it may load content with JavaScript)."))
        return
    try:
        summary = await ai_task(
            "You summarize web pages faithfully in clear plain language, at most 200 words. Use only the supplied text and invent nothing.",
            f"Title: {page['title']}\n\n{page['text']}", max_tokens=450, temperature=0.3)
    except Exception as e:
        summary = f"*(AI summary unavailable: {friendly_ai_error(e)})*"
    embed = base_embed(title=f"📄 {page['title'][:200] or 'Summary'}", description=summary[:4000])
    embed.add_field(name="🔗 Source", value=page["url"][:1020], inline=False)
    await interaction.followup.send(embed=embed)


@ai.command(name="explain", description="Explain a concept, term or teaching in simple words")
@app_commands.describe(topic="What would you like explained?")
@app_commands.checks.cooldown(1, 6.0)
async def ai_explain(interaction: discord.Interaction, topic: str):
    await answer_question(interaction, f"Please explain this clearly for a learner: {topic}", False)


async def verse_autocomplete(interaction: discord.Interaction, current: str):
    cur = current.lower()
    return [app_commands.Choice(name=v["ref"][:100], value=vid) for vid, v in VERSES.items()
            if not cur or cur in v["ref"].lower() or cur in vid][:25]


@ai.command(name="verseexplain", description="Explain a verified verse in simple words")
@app_commands.describe(verse="Pick a verse from the verified library")
@app_commands.autocomplete(verse=verse_autocomplete)
@app_commands.checks.cooldown(1, 8.0)
async def ai_verseexplain(interaction: discord.Interaction, verse: str):
    vid = verse if verse in VERSES else next((k for k, v in VERSES.items() if verse.lower() in v["ref"].lower()), None)
    if vid is None:
        await interaction.response.send_message("🙏 Pick a verse from the suggestions list (verified library only).", ephemeral=True)
        return
    await interaction.response.defer(thinking=True)
    v = VERSES[vid]
    prompt = (f"Explain this verse simply and warmly, with its practical meaning for daily life. Do not quote any other verse.\n"
              f"[{v['ref']}]\n{v['sa']}\nTranslation: {v['en']}")
    try:
        answer = await ai_reply(prompt, [], extra=ASK_EXTRA, max_tokens=600)
    except Exception as e:
        answer = friendly_ai_error(e)
    await interaction.followup.send(embed=verse_embed(vid))
    await send_plain(interaction, answer)


LANGUAGES = ["Hindi", "Tamil", "Telugu", "Kannada", "Malayalam", "Bengali", "Marathi", "Gujarati", "Sanskrit", "English"]


@ai.command(name="translate", description="Translate text into another language")
@app_commands.describe(text="The text to translate", language="Target language")
@app_commands.choices(language=[app_commands.Choice(name=l, value=l) for l in LANGUAGES])
@app_commands.checks.cooldown(1, 5.0)
async def ai_translate(interaction: discord.Interaction, text: str, language: str):
    await interaction.response.defer(thinking=True)
    try:
        out = await ai_task(
            f"You are an accurate translator. Translate the user's text into {language}. Output only the translation, nothing else.",
            text, max_tokens=700, temperature=0.2, scrub=False)
    except Exception as e:
        out = friendly_ai_error(e)
    await send_plain(interaction, out + "\n-# AI translation. Please double-check anything important.")


@ai.command(name="meaning", description="Meaning of a Sanskrit, Hindi or Tamil word")
@app_commands.describe(word="The word to look up")
@app_commands.checks.cooldown(1, 5.0)
async def ai_meaning(interaction: discord.Interaction, word: str):
    await interaction.response.defer(thinking=True)
    try:
        out = await ai_task(
            "You are a careful Sanskrit, Hindi and Tamil lexicon assistant. Give the meaning(s), the root or etymology only if you are sure "
            "(otherwise say you are unsure), and one short example sentence. Under 120 words, plain text, no verses.",
            word, max_tokens=350, temperature=0.3)
    except Exception as e:
        out = friendly_ai_error(e)
    await send_plain(interaction, out)


STORY_SYSTEM = (
    "You retell well-known stories from the Itihasa, Puranas and traditional lore faithfully, in the widely accepted version. "
    "Mention if versions differ, never invent details, and write no Sanskrit or Tamil verses. About 220 words, warm storytelling "
    "tone in plain text, ending with a one-line takeaway."
)


@ai.command(name="story", description="Hear a traditional story (Ramayana, Mahabharata, Puranas...)")
@app_commands.describe(topic="For example: Prahlada, Savitri, Nala and Damayanti, Gajendra Moksha")
@app_commands.checks.cooldown(1, 10.0)
async def ai_story(interaction: discord.Interaction, topic: str):
    await interaction.response.defer(thinking=True)
    try:
        out = await ai_task(STORY_SYSTEM, f"Tell the story of: {topic}", max_tokens=600, temperature=0.6)
    except Exception as e:
        out = friendly_ai_error(e)
    await send_plain(interaction, out)


@ai.command(name="define", description="A short definition of a dharmic or philosophical term")
@app_commands.describe(term="For example: maya, sharanagati, prarabdha, kainkarya")
@app_commands.checks.cooldown(1, 5.0)
async def ai_define(interaction: discord.Interaction, term: str):
    await interaction.response.defer(thinking=True)
    try:
        out = await ai_task(
            "Define the given term from Sanatana Dharma accurately in 2-4 sentences. If schools differ, mention that briefly. "
            "Plain text, no verses.", term, max_tokens=300, temperature=0.3)
    except Exception as e:
        out = friendly_ai_error(e)
    await send_plain(interaction, out)


@ai.command(name="advice", description="Gentle guidance on a life situation through a dharmic lens")
@app_commands.describe(situation="Describe what you're going through")
@app_commands.checks.cooldown(1, 10.0)
async def ai_advice(interaction: discord.Interaction, situation: str):
    await interaction.response.defer(thinking=True, ephemeral=True)
    extra = ("The user shares a personal situation. Respond with warmth and practical perspective (duty, karma, patience, seva) in a few "
             "short paragraphs of plain text. If they mention hopelessness, self-harm or abuse, gently urge them to reach out to a trusted "
             "person or a local emergency or helpline number right away.")
    try:
        out = await ai_reply(situation, [], extra=extra, max_tokens=600)
    except Exception as e:
        out = friendly_ai_error(e)
    await send_plain(interaction, out, ephemeral=True)


STYLES = ["polite", "traditional", "casual", "formal", "short"]


@ai.command(name="rephrase", description="Rewrite your message in a chosen style")
@app_commands.describe(text="Your message", style="Style to use")
@app_commands.choices(style=[app_commands.Choice(name=s.title(), value=s) for s in STYLES])
@app_commands.checks.cooldown(1, 5.0)
async def ai_rephrase(interaction: discord.Interaction, text: str, style: str):
    await interaction.response.defer(thinking=True)
    try:
        out = await ai_task(f"Rewrite the user's message in a {style} style, keeping the meaning and language. Output only the rewritten message.",
                            text, max_tokens=400, temperature=0.6, scrub=False)
    except Exception as e:
        out = friendly_ai_error(e)
    await send_plain(interaction, out)


@ai.command(name="nameideas", description="Traditional name suggestions with meanings")
@app_commands.describe(gender="Boy, Girl or Any", letter="Optional starting letter or syllable")
@app_commands.choices(gender=[app_commands.Choice(name=g, value=g) for g in ["Boy", "Girl", "Any"]])
@app_commands.checks.cooldown(1, 8.0)
async def ai_nameideas(interaction: discord.Interaction, gender: str, letter: Optional[str] = None):
    await interaction.response.defer(thinking=True)
    ask = f"Suggest 10 traditional Hindu names for a {gender.lower()} child" + (f" starting with '{letter}'" if letter else "") + \
          ", each with a one-line meaning. Plain list, only well-established names and meanings."
    try:
        out = await ai_task("You suggest traditional Hindu names with accurate meanings. Never invent meanings; skip a name if unsure.",
                            ask, max_tokens=450, temperature=0.6)
    except Exception as e:
        out = friendly_ai_error(e)
    await send_plain(interaction, out)


@ai.command(name="quiz", description="A short 3-question quiz on a dharma topic")
@app_commands.describe(topic="For example: Ramayana, Upanishads, festivals, Alvars")
@app_commands.checks.cooldown(1, 15.0)
async def ai_quiz(interaction: discord.Interaction, topic: str):
    await interaction.response.defer(thinking=True)
    try:
        out = await ai_task(
            "Create exactly 3 multiple-choice questions (options A-D) on the given Sanatana Dharma topic, using only widely established facts. "
            "At the end add an answer key where each answer is wrapped in Discord spoiler tags, like ||1-B||. No Sanskrit or Tamil verses.",
            topic, max_tokens=500, temperature=0.5)
    except Exception as e:
        out = friendly_ai_error(e)
    await send_plain(interaction, out)


@ai.command(name="forget", description="Clear what I remember from our conversations")
async def ai_forget(interaction: discord.Interaction):
    for k in [k for k in chat_memory if k[1] == interaction.user.id]:
        chat_memory.pop(k, None)
    await interaction.response.send_message("🧹 Done, fresh start!", ephemeral=True)


# ─── /level ───────────────────────────────────────────────────────────────

level = make_group("level", "Word-count leveling, ranks and rewards")
admin_only = app_commands.checks.has_permissions(manage_guild=True)


async def get_levels(guild_id: int, user_id: int):
    cur = await bot.db.execute("SELECT words, level FROM levels WHERE guild_id = ? AND user_id = ?", (guild_id, user_id))
    row = await cur.fetchone()
    await cur.close()
    return (row[0], row[1]) if row else (0, 0)


async def put_levels(guild_id: int, user_id: int, words: int):
    words = max(0, words)
    await bot.db.execute(
        "INSERT INTO levels (guild_id, user_id, words, level) VALUES (?, ?, ?, ?) "
        "ON CONFLICT(guild_id, user_id) DO UPDATE SET words = excluded.words, level = excluded.level",
        (guild_id, user_id, words, words // WORDS_PER_LEVEL))
    await bot.db.commit()
    return words, words // WORDS_PER_LEVEL


async def give_roles_up_to(member: discord.Member, lvl: int):
    for n in range(1, lvl + 1):
        await give_level_role(member, n)


def progress_bar(current: int, total: int, length: int = 14) -> str:
    filled = min(length, int(length * current / max(total, 1)))
    return "█" * filled + "░" * (length - filled)


@level.command(name="rank", description="Show your level and progress")
@app_commands.describe(member="Whose rank to show (default: you)")
async def level_rank(interaction: discord.Interaction, member: Optional[discord.Member] = None):
    member = member or interaction.user
    words, lvl = await get_levels(interaction.guild_id, member.id)
    cur = await bot.db.execute(
        "SELECT COUNT(*) + 1 FROM levels WHERE guild_id = ? AND (level > ? OR (level = ? AND words > ?))",
        (interaction.guild_id, lvl, lvl, words))
    pos = (await cur.fetchone())[0]
    await cur.close()
    into = words % WORDS_PER_LEVEL
    e = base_embed(title=f"🪔 Rank — {member.display_name}")
    e.set_thumbnail(url=member.display_avatar.url)
    e.add_field(name="🏅 Level", value=str(lvl), inline=True)
    e.add_field(name="📊 Server Rank", value=f"#{pos}", inline=True)
    e.add_field(name="📝 Words", value=f"{into} / {WORDS_PER_LEVEL} (total {words})", inline=True)
    e.add_field(name="Progress", value=f"`{progress_bar(into, WORDS_PER_LEVEL)}`", inline=False)
    e.set_footer(text="Hari Om 🙏")
    await interaction.response.send_message(embed=e)


@level.command(name="leaderboard", description="Top 10 members by words written")
async def level_leaderboard(interaction: discord.Interaction):
    cur = await bot.db.execute(
        "SELECT user_id, words, level FROM levels WHERE guild_id = ? ORDER BY level DESC, words DESC LIMIT 10", (interaction.guild_id,))
    rows = await cur.fetchall()
    await cur.close()
    if not rows:
        await interaction.response.send_message(embed=info_embed("📈 Leaderboard", "No one has earned any words yet. Start chatting! 🙏"))
        return
    medals, lines = ["🥇", "🥈", "🥉"], []
    for i, (uid, words, lvl) in enumerate(rows):
        m = interaction.guild.get_member(uid)
        lines.append(f"{medals[i] if i < 3 else f'**{i + 1}.**'} {m.display_name if m else f'User {uid}'}: Level {lvl} ({words} words)")
    await interaction.response.send_message(embed=info_embed(f"📈 Leaderboard — {interaction.guild.name}", "\n".join(lines)))


@level.command(name="setrole", description="Set the role rewarded at a level")
@app_commands.describe(level="Level number", role="Role to give")
@admin_only
async def level_setrole(interaction: discord.Interaction, level: app_commands.Range[int, 1, 100], role: discord.Role):
    await bot.db.execute(
        "INSERT INTO level_roles (guild_id, level, role_id) VALUES (?, ?, ?) ON CONFLICT(guild_id, level) DO UPDATE SET role_id = excluded.role_id",
        (interaction.guild_id, level, role.id))
    await bot.db.commit()
    await interaction.response.send_message(embed=info_embed("✅ Level Role Set", f"Reaching **Level {level}** now gives {role.mention}."))


@level.command(name="removerole", description="Stop rewarding a role at a level")
@app_commands.describe(level="Level number")
@admin_only
async def level_removerole(interaction: discord.Interaction, level: app_commands.Range[int, 1, 100]):
    await bot.db.execute(
        "INSERT INTO level_roles (guild_id, level, role_id) VALUES (?, ?, 0) ON CONFLICT(guild_id, level) DO UPDATE SET role_id = 0",
        (interaction.guild_id, level))
    await bot.db.commit()
    await interaction.response.send_message(embed=info_embed("✅ Level Role Removed", f"No role will be given at **Level {level}** anymore."))


@level.command(name="roles", description="List the level reward roles")
async def level_roles(interaction: discord.Interaction):
    cur = await bot.db.execute("SELECT level, role_id FROM level_roles WHERE guild_id = ?", (interaction.guild_id,))
    overrides = dict(await cur.fetchall())
    await cur.close()
    merged = {**LEVEL_ROLES, **overrides}
    lines = [f"**Level {lv}** ({lv * WORDS_PER_LEVEL} words): " + (f"<@&{rid}>" if rid else "*(none)*") for lv, rid in sorted(merged.items())]
    await interaction.response.send_message(embed=info_embed("🎖️ Level Roles", "\n".join(lines) or "No level roles set."),
                                            allowed_mentions=discord.AllowedMentions.none())


@level.command(name="addwords", description="Add words to a member's total")
@app_commands.describe(member="Member", amount="Words to add")
@admin_only
async def level_addwords(interaction: discord.Interaction, member: discord.Member, amount: app_commands.Range[int, 1, 1000000]):
    words, lvl = await get_levels(interaction.guild_id, member.id)
    words, new_lvl = await put_levels(interaction.guild_id, member.id, words + amount)
    await give_roles_up_to(member, new_lvl)
    await interaction.response.send_message(embed=info_embed("✅ Words Added", f"{member.mention} is now **Level {new_lvl}** with **{words}** words."))


@level.command(name="removewords", description="Remove words from a member's total")
@app_commands.describe(member="Member", amount="Words to remove")
@admin_only
async def level_removewords(interaction: discord.Interaction, member: discord.Member, amount: app_commands.Range[int, 1, 1000000]):
    words, lvl = await get_levels(interaction.guild_id, member.id)
    words, new_lvl = await put_levels(interaction.guild_id, member.id, words - amount)
    await interaction.response.send_message(embed=info_embed(
        "✅ Words Removed", f"{member.mention} is now **Level {new_lvl}** with **{words}** words.\n*(Roles already earned are not removed automatically.)*"))


@level.command(name="setwords", description="Set a member's exact word total")
@app_commands.describe(member="Member", total="Exact word total")
@admin_only
async def level_setwords(interaction: discord.Interaction, member: discord.Member, total: app_commands.Range[int, 0, 10000000]):
    words, new_lvl = await put_levels(interaction.guild_id, member.id, total)
    await give_roles_up_to(member, new_lvl)
    await interaction.response.send_message(embed=info_embed("✅ Words Set", f"{member.mention} is now **Level {new_lvl}** with **{words}** words."))


@level.command(name="reset", description="Reset one member's level and words")
@app_commands.describe(member="Member to reset")
@admin_only
async def level_reset(interaction: discord.Interaction, member: discord.Member):
    await bot.db.execute("DELETE FROM levels WHERE guild_id = ? AND user_id = ?", (interaction.guild_id, member.id))
    await bot.db.commit()
    await interaction.response.send_message(embed=info_embed("🔄 Reset", f"{member.mention}'s level and words were reset."))


@level.command(name="resetall", description="Reset EVERYONE's levels (type CONFIRM to proceed)")
@app_commands.describe(confirm="Type CONFIRM in capital letters")
@admin_only
async def level_resetall(interaction: discord.Interaction, confirm: str):
    if confirm != "CONFIRM":
        await interaction.response.send_message("Type `CONFIRM` (capital letters) to really reset everyone.", ephemeral=True)
        return
    await bot.db.execute("DELETE FROM levels WHERE guild_id = ?", (interaction.guild_id,))
    await bot.db.commit()
    await interaction.response.send_message(embed=info_embed("🔄 All Levels Reset", "Every member's levels and words were cleared."))


# ─── /config ──────────────────────────────────────────────────────────────

config = make_group("config", "Server setup: channels, messages and auto-role")


def add_channel_setter(name: str, key: str, label: str, desc: str):
    async def cb(interaction: discord.Interaction, channel: discord.TextChannel):
        await set_setting(interaction.guild_id, key, channel.id)
        await interaction.response.send_message(embed=info_embed("✅ Channel Set", f"{label} → {channel.mention}"))
    cb.__name__ = cb.__qualname__ = f"config_{name}"
    cb = app_commands.describe(channel="The channel to use")(cb)
    cb = admin_only(cb)
    config.add_command(app_commands.Command(name=name, description=desc[:100], callback=cb))


add_channel_setter("setwelcomechannel", "welcome_channel", "Welcome messages", "Set the channel for welcome messages")
add_channel_setter("setgoodbyechannel", "goodbye_channel", "Goodbye messages", "Set the channel for goodbye messages (separate from welcome)")
add_channel_setter("setlogchannel", "log_channel", "Staff logs", "Set the channel for staff logs")
add_channel_setter("setverifychannel", "verify_channel", "Verification panel", "Set the channel where the ticket panel is posted")
add_channel_setter("setchatchannel", "chat_channel", "AI chat channel (I reply to every message there)", "Set a channel where I chat with everyone like a person")


@config.command(name="clearchatchannel", description="Stop replying to every message in the chat channel")
@admin_only
async def config_clearchatchannel(interaction: discord.Interaction):
    await del_setting(interaction.guild_id, "chat_channel")
    await interaction.response.send_message(embed=info_embed("✅ Chat Channel Cleared", "I'll only reply when mentioned or replied to."))


@config.command(name="setwelcomemessage", description="Customize the welcome text ({mention} {name} {server} {count})")
@app_commands.describe(text="Welcome text. Placeholders: {mention} {name} {server} {count}")
@admin_only
async def config_setwelcomemessage(interaction: discord.Interaction, text: app_commands.Range[str, 1, 3500]):
    await set_setting(interaction.guild_id, "welcome_message", text)
    await interaction.response.send_message(embed=info_embed("✅ Welcome Message Saved", "Use `/config testwelcome` to preview it."), ephemeral=True)


@config.command(name="setgoodbyemessage", description="Customize the goodbye text ({mention} {name} {server} {count})")
@app_commands.describe(text="Goodbye text. Placeholders: {mention} {name} {server} {count}")
@admin_only
async def config_setgoodbyemessage(interaction: discord.Interaction, text: app_commands.Range[str, 1, 3500]):
    await set_setting(interaction.guild_id, "goodbye_message", text)
    await interaction.response.send_message(embed=info_embed("✅ Goodbye Message Saved", "Use `/config testgoodbye` to preview it."), ephemeral=True)


@config.command(name="resetwelcomemessage", description="Go back to the default welcome message")
@admin_only
async def config_resetwelcomemessage(interaction: discord.Interaction):
    await del_setting(interaction.guild_id, "welcome_message")
    await interaction.response.send_message(embed=info_embed("✅ Reset", "The default welcome message is back."), ephemeral=True)


@config.command(name="resetgoodbyemessage", description="Go back to the default goodbye message")
@admin_only
async def config_resetgoodbyemessage(interaction: discord.Interaction):
    await del_setting(interaction.guild_id, "goodbye_message")
    await interaction.response.send_message(embed=info_embed("✅ Reset", "The default goodbye message is back."), ephemeral=True)


@config.command(name="testwelcome", description="Preview the welcome message (only you can see it)")
@admin_only
async def config_testwelcome(interaction: discord.Interaction):
    await interaction.response.send_message(embed=await build_welcome_embed(interaction.user), ephemeral=True)


@config.command(name="testgoodbye", description="Preview the goodbye message (only you can see it)")
@admin_only
async def config_testgoodbye(interaction: discord.Interaction):
    await interaction.response.send_message(embed=await build_goodbye_embed(interaction.user), ephemeral=True)


@config.command(name="setautorole", description="Give this role to everyone who joins")
@app_commands.describe(role="Role to give on join")
@admin_only
async def config_setautorole(interaction: discord.Interaction, role: discord.Role):
    await set_setting(interaction.guild_id, "autorole", role.id)
    await interaction.response.send_message(embed=info_embed("✅ Auto-role Set", f"New members will receive {role.mention}."),
                                            allowed_mentions=discord.AllowedMentions.none())


@config.command(name="clearautorole", description="Stop giving a role on join")
@admin_only
async def config_clearautorole(interaction: discord.Interaction):
    await del_setting(interaction.guild_id, "autorole")
    await interaction.response.send_message(embed=info_embed("✅ Auto-role Cleared", "New members won't get an automatic role."))


@config.command(name="show", description="Show the current server configuration")
@admin_only
async def config_show(interaction: discord.Interaction):
    g = interaction.guild

    async def chan(key, default):
        ch = await channel_setting(g, key, default)
        return ch.mention if ch else "*(not set)*"

    autorole = await get_setting(g.id, "autorole")
    chat = await get_setting(g.id, "chat_channel")
    e = base_embed(title="⚙️ Server Configuration")
    e.add_field(name="Welcome channel", value=await chan("welcome_channel", WELCOME_CHANNEL_ID), inline=True)
    e.add_field(name="Goodbye channel", value=await chan("goodbye_channel", GOODBYE_CHANNEL_ID), inline=True)
    e.add_field(name="Log channel", value=await chan("log_channel", LOG_CHANNEL_ID), inline=True)
    e.add_field(name="Verify channel", value=await chan("verify_channel", VERIFY_CHANNEL_ID), inline=True)
    e.add_field(name="AI chat channel", value=f"<#{chat}>" if chat else "*(only mentions/replies)*", inline=True)
    e.add_field(name="Auto-role", value=f"<@&{autorole}>" if autorole else "*(none)*", inline=True)
    e.add_field(name="Custom welcome text", value="yes" if await get_setting(g.id, "welcome_message") else "default", inline=True)
    e.add_field(name="Custom goodbye text", value="yes" if await get_setting(g.id, "goodbye_message") else "default", inline=True)
    e.add_field(name="Automod", value=await get_setting(g.id, "automod", "on"), inline=True)
    await interaction.response.send_message(embed=e, ephemeral=True, allowed_mentions=discord.AllowedMentions.none())

# ─── /reactionrole ────────────────────────────────────────────────────────

reactionrole = make_group("reactionrole", "Create and manage emoji reaction roles")


@reactionrole.command(name="add", description="Attach an emoji reaction to a message to grant a role")
@app_commands.describe(channel="Channel containing the role message", message_id="ID of the message", emoji="Unicode emoji", role="Role granted when reacted")
@app_commands.checks.has_permissions(manage_roles=True, manage_messages=True)
@app_commands.checks.bot_has_permissions(manage_roles=True, add_reactions=True, read_message_history=True)
async def reactionrole_add(interaction: discord.Interaction, channel: discord.TextChannel, message_id: str, emoji: str, role: discord.Role):
    if not message_id.isdigit():
        return await deny(interaction, "Message ID must contain digits only.")
    block = role_block(interaction, role)
    if block:
        return await deny(interaction, block)
    try:
        message = await channel.fetch_message(int(message_id))
        await message.add_reaction(emoji)
    except discord.NotFound:
        return await deny(interaction, "I couldn't find that message in the selected channel.")
    except discord.HTTPException:
        return await deny(interaction, "That emoji is invalid or I cannot add it to the message.")
    await bot.db.execute(
        "INSERT INTO reaction_roles (guild_id, channel_id, message_id, emoji, role_id) VALUES (?, ?, ?, ?, ?) "
        "ON CONFLICT(guild_id, message_id, emoji) DO UPDATE SET role_id=excluded.role_id, channel_id=excluded.channel_id",
        (interaction.guild_id, channel.id, int(message_id), emoji, role.id),
    )
    await bot.db.commit()
    await interaction.response.send_message(embed=info_embed("✅ Reaction Role Added", f"React with {emoji} to [this message]({message.jump_url}) to receive {role.mention}. Remove your reaction to give it up."), ephemeral=True)


@reactionrole.command(name="remove", description="Remove a reaction-role mapping")
@app_commands.describe(message_id="ID of the role message", emoji="The same emoji used when creating it")
@app_commands.checks.has_permissions(manage_roles=True, manage_messages=True)
async def reactionrole_remove(interaction: discord.Interaction, message_id: str, emoji: str):
    if not message_id.isdigit():
        return await deny(interaction, "Message ID must contain digits only.")
    cur = await bot.db.execute("DELETE FROM reaction_roles WHERE guild_id=? AND message_id=? AND emoji=?", (interaction.guild_id, int(message_id), emoji))
    await bot.db.commit()
    await interaction.response.send_message("✅ Reaction-role mapping removed." if cur.rowcount else "No matching mapping was found.", ephemeral=True)


@reactionrole.command(name="list", description="List this server's reaction-role mappings")
@app_commands.checks.has_permissions(manage_roles=True)
async def reactionrole_list(interaction: discord.Interaction):
    cur = await bot.db.execute("SELECT channel_id, message_id, emoji, role_id FROM reaction_roles WHERE guild_id=? ORDER BY message_id", (interaction.guild_id,))
    rows = await cur.fetchall()
    await cur.close()
    if not rows:
        return await interaction.response.send_message("No reaction roles have been configured yet.", ephemeral=True)
    lines = [f"{emoji} → <@&{role_id}> · <#{channel_id}> · message `{message_id}`" for channel_id, message_id, emoji, role_id in rows]
    await interaction.response.send_message(embed=info_embed(f"🎭 Reaction Roles ({len(rows)})", "\n".join(lines)[:4000]), ephemeral=True, allowed_mentions=discord.AllowedMentions.none())


# ─── /mod ─────────────────────────────────────────────────────────────────

mod = make_group("mod", "Moderation tools")


def action_embed(title: str, description: str, reason: Optional[str], moderator: discord.abc.User) -> discord.Embed:
    e = base_embed(title=title, description=description)
    e.add_field(name="📝 Reason", value=(reason or "No reason provided")[:1000], inline=False)
    e.add_field(name="⚖️ Moderator", value=moderator.mention, inline=False)
    return e


def hierarchy_block(interaction: discord.Interaction, target: discord.Member, allow_self: bool = False) -> Optional[str]:
    guild = interaction.guild
    if target.id == interaction.user.id and not allow_self:
        return "You can't do that to yourself."
    if target.id == guild.owner_id:
        return "You can't target the server owner."
    if target.id == bot.user.id:
        return "I'm not going to do that to myself 🙏"
    if interaction.user.id != guild.owner_id and target.id != interaction.user.id and target.top_role >= interaction.user.top_role:
        return "That member's top role is equal to or higher than yours."
    if target.top_role >= guild.me.top_role:
        return "That member's top role is higher than mine, so I can't act on them."
    return None


async def deny(interaction: discord.Interaction, text: str):
    await interaction.response.send_message(f"🚫 {text}", ephemeral=True)


async def done(interaction: discord.Interaction, embed: discord.Embed):
    await interaction.response.send_message(embed=embed)
    await log_event(interaction.guild, embed)


@mod.command(name="kick", description="Kick a member")
@app_commands.describe(member="Member to kick", reason="Why")
@app_commands.checks.has_permissions(kick_members=True)
@app_commands.checks.bot_has_permissions(kick_members=True)
async def mod_kick(interaction: discord.Interaction, member: discord.Member, reason: Optional[str] = None):
    block = hierarchy_block(interaction, member)
    if block:
        return await deny(interaction, block)
    await member.kick(reason=f"{interaction.user}: {reason or 'No reason'}")
    await done(interaction, action_embed("👢 Member Kicked", f"**{member}** has been kicked.", reason, interaction.user))


@mod.command(name="ban", description="Ban a member")
@app_commands.describe(member="Member to ban", reason="Why")
@app_commands.checks.has_permissions(ban_members=True)
@app_commands.checks.bot_has_permissions(ban_members=True)
async def mod_ban(interaction: discord.Interaction, member: discord.Member, reason: Optional[str] = None):
    block = hierarchy_block(interaction, member)
    if block:
        return await deny(interaction, block)
    await member.ban(reason=f"{interaction.user}: {reason or 'No reason'}")
    await done(interaction, action_embed("⚔️ Member Banned", f"**{member}** has been banned.", reason, interaction.user))


@mod.command(name="unban", description="Unban a user by their ID")
@app_commands.describe(user_id="The user's numeric ID", reason="Why")
@app_commands.checks.has_permissions(ban_members=True)
@app_commands.checks.bot_has_permissions(ban_members=True)
async def mod_unban(interaction: discord.Interaction, user_id: str, reason: Optional[str] = None):
    if not user_id.strip().isdigit():
        return await deny(interaction, "Give me the numeric user ID.")
    try:
        user = await bot.fetch_user(int(user_id))
        await interaction.guild.unban(user, reason=f"{interaction.user}: {reason or 'No reason'}")
    except discord.NotFound:
        return await deny(interaction, "I couldn't find a banned user with that ID.")
    await done(interaction, action_embed("🕊️ Member Unbanned", f"**{user}** has been unbanned.", reason, interaction.user))


@mod.command(name="mute", description="Timeout a member for some minutes")
@app_commands.describe(member="Member to mute", minutes="Duration in minutes (max 40320 = 28 days)", reason="Why")
@app_commands.checks.has_permissions(moderate_members=True)
@app_commands.checks.bot_has_permissions(moderate_members=True)
async def mod_mute(interaction: discord.Interaction, member: discord.Member, minutes: app_commands.Range[int, 1, 40320], reason: Optional[str] = None):
    block = hierarchy_block(interaction, member)
    if block:
        return await deny(interaction, block)
    await member.timeout(discord.utils.utcnow() + datetime.timedelta(minutes=minutes), reason=f"{interaction.user}: {reason or 'No reason'}")
    await done(interaction, action_embed("🤐 Member Muted", f"**{member}** muted for **{minutes} minute(s)**.", reason, interaction.user))


@mod.command(name="unmute", description="Remove a member's timeout")
@app_commands.describe(member="Member to unmute")
@app_commands.checks.has_permissions(moderate_members=True)
@app_commands.checks.bot_has_permissions(moderate_members=True)
async def mod_unmute(interaction: discord.Interaction, member: discord.Member):
    await member.timeout(None, reason=f"Unmuted by {interaction.user}")
    await done(interaction, action_embed("🔊 Member Unmuted", f"**{member}** may speak again. 🙏", None, interaction.user))


@mod.command(name="warn", description="Record a warning for a member")
@app_commands.describe(member="Member to warn", reason="Why")
@app_commands.checks.has_permissions(moderate_members=True)
async def mod_warn(interaction: discord.Interaction, member: discord.Member, reason: Optional[str] = None):
    await bot.db.execute(
        "INSERT INTO warnings (guild_id, user_id, moderator_id, reason, created_at) VALUES (?, ?, ?, ?, ?)",
        (interaction.guild_id, member.id, interaction.user.id, reason or "No reason provided", discord.utils.utcnow().isoformat()))
    await bot.db.commit()
    cur = await bot.db.execute("SELECT COUNT(*) FROM warnings WHERE guild_id = ? AND user_id = ?", (interaction.guild_id, member.id))
    total = (await cur.fetchone())[0]
    await cur.close()
    await done(interaction, action_embed("⚠️ Warning Recorded", f"{member.mention} now has **{total}** warning(s).", reason, interaction.user))


@mod.command(name="warnings", description="List a member's warnings")
@app_commands.describe(member="Member (default: you)")
@app_commands.checks.has_permissions(moderate_members=True)
async def mod_warnings(interaction: discord.Interaction, member: Optional[discord.Member] = None):
    member = member or interaction.user
    cur = await bot.db.execute(
        "SELECT id, moderator_id, reason, created_at FROM warnings WHERE guild_id = ? AND user_id = ? ORDER BY id DESC LIMIT 15",
        (interaction.guild_id, member.id))
    rows = await cur.fetchall()
    await cur.close()
    if not rows:
        return await interaction.response.send_message(embed=info_embed(f"📋 Warnings — {member.display_name}", "No warnings. A clean record. 🙏"))
    lines = [f"`#{wid}` **{reason}** · by <@{mid}> · {created[:10]}" for wid, mid, reason, created in rows]
    await interaction.response.send_message(embed=info_embed(f"📋 Warnings — {member.display_name} ({len(rows)})", "\n".join(lines)),
                                            allowed_mentions=discord.AllowedMentions.none())


@mod.command(name="delwarn", description="Delete one warning by its number")
@app_commands.describe(warn_id="The warning number shown in /mod warnings")
@app_commands.checks.has_permissions(moderate_members=True)
async def mod_delwarn(interaction: discord.Interaction, warn_id: int):
    cur = await bot.db.execute("DELETE FROM warnings WHERE id = ? AND guild_id = ?", (warn_id, interaction.guild_id))
    await bot.db.commit()
    if cur.rowcount:
        await done(interaction, action_embed("🗑️ Warning Deleted", f"Warning `#{warn_id}` removed.", None, interaction.user))
    else:
        await deny(interaction, "No warning with that number in this server.")


@mod.command(name="clearwarnings", description="Delete all of a member's warnings")
@app_commands.describe(member="Member")
@app_commands.checks.has_permissions(moderate_members=True)
async def mod_clearwarnings(interaction: discord.Interaction, member: discord.Member):
    await bot.db.execute("DELETE FROM warnings WHERE guild_id = ? AND user_id = ?", (interaction.guild_id, member.id))
    await bot.db.commit()
    await done(interaction, action_embed("🧹 Warnings Cleared", f"All warnings for {member.mention} were removed.", None, interaction.user))


@mod.command(name="purge", description="Delete the most recent messages in this channel")
@app_commands.describe(count="How many messages (1-100)")
@app_commands.checks.has_permissions(manage_messages=True)
@app_commands.checks.bot_has_permissions(manage_messages=True, read_message_history=True)
async def mod_purge(interaction: discord.Interaction, count: app_commands.Range[int, 1, 100]):
    await interaction.response.defer(ephemeral=True)
    deleted = await interaction.channel.purge(limit=count)
    e = action_embed("🧹 Messages Purged", f"Removed **{len(deleted)}** message(s) from {interaction.channel.mention}.", None, interaction.user)
    await interaction.followup.send(embed=e, ephemeral=True)
    await log_event(interaction.guild, e)


@mod.command(name="purgeuser", description="Delete a member's recent messages in this channel")
@app_commands.describe(member="Whose messages", count="How many of their messages (1-100)")
@app_commands.checks.has_permissions(manage_messages=True)
@app_commands.checks.bot_has_permissions(manage_messages=True, read_message_history=True)
async def mod_purgeuser(interaction: discord.Interaction, member: discord.Member, count: app_commands.Range[int, 1, 100]):
    await interaction.response.defer(ephemeral=True)
    counter = {"n": 0}

    def check(m: discord.Message) -> bool:
        if m.author.id == member.id and counter["n"] < count:
            counter["n"] += 1
            return True
        return False

    deleted = await interaction.channel.purge(limit=500, check=check)
    e = action_embed("🧹 User Messages Purged", f"Removed **{len(deleted)}** message(s) by {member.mention} in {interaction.channel.mention}.", None, interaction.user)
    await interaction.followup.send(embed=e, ephemeral=True)
    await log_event(interaction.guild, e)


@mod.command(name="slowmode", description="Set slowmode for a channel (0 turns it off)")
@app_commands.describe(seconds="Seconds between messages (0-21600)", channel="Channel (default: this one)")
@app_commands.checks.has_permissions(manage_channels=True)
@app_commands.checks.bot_has_permissions(manage_channels=True)
async def mod_slowmode(interaction: discord.Interaction, seconds: app_commands.Range[int, 0, 21600], channel: Optional[discord.TextChannel] = None):
    target = channel or interaction.channel
    await target.edit(slowmode_delay=seconds, reason=f"Slowmode by {interaction.user}")
    await done(interaction, action_embed("🐢 Slowmode Updated", f"{target.mention}: **{seconds}s**" if seconds else f"Slowmode turned off in {target.mention}.", None, interaction.user))


async def _set_everyone(interaction: discord.Interaction, channel: Optional[discord.TextChannel], title: str, text: str, **perms):
    target = channel or interaction.channel
    ov = target.overwrites_for(interaction.guild.default_role)
    for k, v in perms.items():
        setattr(ov, k, v)
    await target.set_permissions(interaction.guild.default_role, overwrite=ov, reason=f"{title} by {interaction.user}")
    await done(interaction, action_embed(title, text.format(c=target.mention), None, interaction.user))


@mod.command(name="lock", description="Stop @everyone from sending messages in a channel")
@app_commands.describe(channel="Channel (default: this one)")
@app_commands.checks.has_permissions(manage_channels=True)
@app_commands.checks.bot_has_permissions(manage_channels=True)
async def mod_lock(interaction: discord.Interaction, channel: Optional[discord.TextChannel] = None):
    await _set_everyone(interaction, channel, "🔒 Channel Locked", "{c} is locked.", send_messages=False)


@mod.command(name="unlock", description="Let @everyone send messages in a channel again")
@app_commands.describe(channel="Channel (default: this one)")
@app_commands.checks.has_permissions(manage_channels=True)
@app_commands.checks.bot_has_permissions(manage_channels=True)
async def mod_unlock(interaction: discord.Interaction, channel: Optional[discord.TextChannel] = None):
    await _set_everyone(interaction, channel, "🔓 Channel Unlocked", "{c} is unlocked.", send_messages=None)


@mod.command(name="hide", description="Hide a channel from @everyone")
@app_commands.describe(channel="Channel (default: this one)")
@app_commands.checks.has_permissions(manage_channels=True)
@app_commands.checks.bot_has_permissions(manage_channels=True)
async def mod_hide(interaction: discord.Interaction, channel: Optional[discord.TextChannel] = None):
    await _set_everyone(interaction, channel, "🙈 Channel Hidden", "{c} is hidden from @everyone.", view_channel=False)


@mod.command(name="unhide", description="Make a hidden channel visible to @everyone again")
@app_commands.describe(channel="Channel (default: this one)")
@app_commands.checks.has_permissions(manage_channels=True)
@app_commands.checks.bot_has_permissions(manage_channels=True)
async def mod_unhide(interaction: discord.Interaction, channel: Optional[discord.TextChannel] = None):
    await _set_everyone(interaction, channel, "👁️ Channel Visible", "{c} is visible again.", view_channel=None)


@mod.command(name="nick", description="Change or clear a member's nickname")
@app_commands.describe(member="Member", nickname="New nickname (leave empty to clear)")
@app_commands.checks.has_permissions(manage_nicknames=True)
@app_commands.checks.bot_has_permissions(manage_nicknames=True)
async def mod_nick(interaction: discord.Interaction, member: discord.Member, nickname: Optional[str] = None):
    block = hierarchy_block(interaction, member, allow_self=True)
    if block:
        return await deny(interaction, block)
    await member.edit(nick=nickname, reason=f"Nickname changed by {interaction.user}")
    await done(interaction, action_embed("✏️ Nickname Updated", f"{member.mention} → **{nickname or 'cleared'}**", None, interaction.user))


def role_block(interaction: discord.Interaction, role: discord.Role) -> Optional[str]:
    if role.managed or role.is_default():
        return "That role can't be assigned manually."
    if interaction.user.id != interaction.guild.owner_id and role >= interaction.user.top_role:
        return "That role is equal to or higher than your top role."
    if role >= interaction.guild.me.top_role:
        return "That role is higher than my top role, so I can't hand it out."
    return None


@mod.command(name="addrole", description="Give a role to a member")
@app_commands.describe(member="Member", role="Role to give")
@app_commands.checks.has_permissions(manage_roles=True)
@app_commands.checks.bot_has_permissions(manage_roles=True)
async def mod_addrole(interaction: discord.Interaction, member: discord.Member, role: discord.Role):
    block = role_block(interaction, role)
    if block:
        return await deny(interaction, block)
    await member.add_roles(role, reason=f"By {interaction.user}")
    await done(interaction, action_embed("🎭 Role Added", f"{role.mention} → {member.mention}", None, interaction.user))


@mod.command(name="removerole", description="Take a role from a member")
@app_commands.describe(member="Member", role="Role to remove")
@app_commands.checks.has_permissions(manage_roles=True)
@app_commands.checks.bot_has_permissions(manage_roles=True)
async def mod_removerole(interaction: discord.Interaction, member: discord.Member, role: discord.Role):
    block = role_block(interaction, role)
    if block:
        return await deny(interaction, block)
    await member.remove_roles(role, reason=f"By {interaction.user}")
    await done(interaction, action_embed("🎭 Role Removed", f"{role.mention} ✕ {member.mention}", None, interaction.user))


@mod.command(name="banlist", description="Show the most recent bans")
@app_commands.checks.has_permissions(ban_members=True)
@app_commands.checks.bot_has_permissions(ban_members=True)
async def mod_banlist(interaction: discord.Interaction):
    await interaction.response.defer(ephemeral=True)
    bans = [entry async for entry in interaction.guild.bans(limit=25)]
    if not bans:
        return await interaction.followup.send(embed=info_embed("🔨 Ban List", "Nobody is banned."), ephemeral=True)
    lines = [f"**{b.user}** (`{b.user.id}`): {(b.reason or 'no reason')[:60]}" for b in bans]
    await interaction.followup.send(embed=info_embed(f"🔨 Ban List ({len(bans)})", "\n".join(lines)), ephemeral=True)


# ─── /automod ─────────────────────────────────────────────────────────────

automod = make_group("automod", "Auto-delete messages containing banned words (any language)")
manage_only = app_commands.checks.has_permissions(manage_guild=True)


@automod.command(name="addword", description="Add a word or phrase to the banned list")
@app_commands.describe(word="Word or phrase (matched anywhere in a message, any language/script)")
@manage_only
async def automod_addword(interaction: discord.Interaction, word: app_commands.Range[str, 2, 100]):
    await bot.db.execute("INSERT OR IGNORE INTO banned_words (guild_id, word) VALUES (?, ?)", (interaction.guild_id, word.lower().strip()))
    await bot.db.commit()
    await interaction.response.send_message("✅ Added. Messages containing it will now be removed automatically.", ephemeral=True)


@automod.command(name="removeword", description="Remove a word or phrase from the banned list")
@app_commands.describe(word="The exact word or phrase to remove")
@manage_only
async def automod_removeword(interaction: discord.Interaction, word: str):
    cur = await bot.db.execute("DELETE FROM banned_words WHERE guild_id = ? AND word = ?", (interaction.guild_id, word.lower().strip()))
    await bot.db.commit()
    await interaction.response.send_message("✅ Removed." if cur.rowcount else "That word wasn't on the list.", ephemeral=True)


@automod.command(name="listwords", description="Show the banned list (only you can see it)")
@manage_only
async def automod_listwords(interaction: discord.Interaction):
    cur = await bot.db.execute("SELECT word FROM banned_words WHERE guild_id = ? ORDER BY word", (interaction.guild_id,))
    rows = await cur.fetchall()
    await cur.close()
    text = ", ".join(w for (w,) in rows) or "The list is empty. Add words with `/automod addword`."
    await interaction.response.send_message(embed=info_embed(f"🚫 Banned Words ({len(rows)})", text[:4000]), ephemeral=True)


@automod.command(name="toggle", description="Turn the automod word filter on or off")
@app_commands.describe(state="on or off")
@app_commands.choices(state=[app_commands.Choice(name="on", value="on"), app_commands.Choice(name="off", value="off")])
@manage_only
async def automod_toggle(interaction: discord.Interaction, state: str):
    await set_setting(interaction.guild_id, "automod", state)
    await interaction.response.send_message(embed=info_embed("🛡️ Automod", f"Word filter is now **{state}**."))


# ─── /info ────────────────────────────────────────────────────────────────

info = make_group("info", "Information about members, the server and the bot")


def total_commands() -> int:
    return sum(len(c.commands) if isinstance(c, app_commands.Group) else 1 for c in bot.tree.get_commands())


def human_duration(seconds: float) -> str:
    d, rem = divmod(int(seconds), 86400)
    h, rem = divmod(rem, 3600)
    m, s = divmod(rem, 60)
    return " ".join(p for p in (f"{d}d" if d else "", f"{h}h" if h else "", f"{m}m" if m else "", f"{s}s") if p)


@info.command(name="ping", description="Check my response speed")
async def info_ping(interaction: discord.Interaction):
    await interaction.response.send_message(embed=info_embed("🏓 Pong!", f"Latency: **{round(bot.latency * 1000)} ms**"))


@info.command(name="botinfo", description="About me")
async def info_botinfo(interaction: discord.Interaction):
    e = base_embed(title="🕉️ Sanatana Margadarshak")
    e.set_thumbnail(url=bot.user.display_avatar.url)
    e.add_field(name="Servers", value=str(len(bot.guilds)), inline=True)
    e.add_field(name="Members", value=str(sum(g.member_count or 0 for g in bot.guilds)), inline=True)
    e.add_field(name="Commands", value=str(total_commands()), inline=True)
    e.add_field(name="Uptime", value=human_duration(time.time() - bot.start_time), inline=True)
    e.add_field(name="Python", value=platform.python_version(), inline=True)
    e.add_field(name="discord.py", value=discord.__version__, inline=True)
    e.set_footer(text="Hari Om 🙏")
    await interaction.response.send_message(embed=e)


@info.command(name="uptime", description="How long I've been running")
async def info_uptime(interaction: discord.Interaction):
    await interaction.response.send_message(embed=info_embed("⏱️ Uptime", human_duration(time.time() - bot.start_time)))


@info.command(name="userinfo", description="Details about a member")
@app_commands.describe(member="Member (default: you)")
async def info_userinfo(interaction: discord.Interaction, member: Optional[discord.Member] = None):
    member = member or interaction.user
    roles = [r.mention for r in reversed(member.roles) if not r.is_default()]
    e = base_embed(title=f"👤 {member.display_name}")
    e.set_thumbnail(url=member.display_avatar.url)
    e.add_field(name="🆔 ID", value=str(member.id), inline=True)
    e.add_field(name="🏷️ Username", value=str(member), inline=True)
    e.add_field(name="🤖 Bot", value="Yes" if member.bot else "No", inline=True)
    e.add_field(name="📅 Account created", value=discord.utils.format_dt(member.created_at, "F"), inline=False)
    if member.joined_at:
        e.add_field(name="📥 Joined server", value=discord.utils.format_dt(member.joined_at, "F"), inline=False)
    e.add_field(name=f"🎭 Roles ({len(roles)})", value=(" ".join(roles) or "None")[:1020], inline=False)
    await interaction.response.send_message(embed=e, allowed_mentions=discord.AllowedMentions.none())


@info.command(name="serverinfo", description="Details about this server")
async def info_serverinfo(interaction: discord.Interaction):
    g = interaction.guild
    e = base_embed(title=f"🏛️ {g.name}")
    if g.icon:
        e.set_thumbnail(url=g.icon.url)
    e.add_field(name="👑 Owner", value=f"<@{g.owner_id}>", inline=True)
    e.add_field(name="🆔 ID", value=str(g.id), inline=True)
    e.add_field(name="🌐 Locale", value=str(g.preferred_locale), inline=True)
    e.add_field(name="👥 Members", value=str(g.member_count), inline=True)
    e.add_field(name="💬 Channels", value=f"Text {len(g.text_channels)} · Voice {len(g.voice_channels)}", inline=True)
    e.add_field(name="🎭 Roles", value=str(len(g.roles)), inline=True)
    e.add_field(name="📅 Created", value=discord.utils.format_dt(g.created_at, "F"), inline=False)
    await interaction.response.send_message(embed=e, allowed_mentions=discord.AllowedMentions.none())


@info.command(name="avatar", description="Show a member's avatar")
@app_commands.describe(member="Member (default: you)")
async def info_avatar(interaction: discord.Interaction, member: Optional[discord.Member] = None):
    member = member or interaction.user
    e = base_embed(title=f"🖼️ {member.display_name}'s avatar")
    e.set_image(url=member.display_avatar.url)
    await interaction.response.send_message(embed=e)


@info.command(name="banner", description="Show a member's profile banner")
@app_commands.describe(member="Member (default: you)")
async def info_banner(interaction: discord.Interaction, member: Optional[discord.Member] = None):
    member = member or interaction.user
    user = await bot.fetch_user(member.id)
    if not user.banner:
        return await interaction.response.send_message(f"{member.display_name} has no profile banner.", ephemeral=True)
    e = base_embed(title=f"🖼️ {member.display_name}'s banner")
    e.set_image(url=user.banner.url)
    await interaction.response.send_message(embed=e)


@info.command(name="roleinfo", description="Details about a role")
@app_commands.describe(role="The role")
async def info_roleinfo(interaction: discord.Interaction, role: discord.Role):
    key = ["administrator", "manage_guild", "manage_roles", "manage_channels", "kick_members", "ban_members", "manage_messages", "moderate_members"]
    perms = ", ".join(p.replace("_", " ") for p in key if getattr(role.permissions, p)) or "none of the key permissions"
    e = base_embed(title=f"🎭 {role.name}")
    e.add_field(name="🆔 ID", value=str(role.id), inline=True)
    e.add_field(name="👥 Members", value=str(len(role.members)), inline=True)
    e.add_field(name="📊 Position", value=str(role.position), inline=True)
    e.add_field(name="🎨 Colour", value=str(role.colour), inline=True)
    e.add_field(name="📣 Mentionable", value="Yes" if role.mentionable else "No", inline=True)
    e.add_field(name="📅 Created", value=discord.utils.format_dt(role.created_at, "D"), inline=True)
    e.add_field(name="🔑 Key permissions", value=perms[:1000], inline=False)
    await interaction.response.send_message(embed=e)


@info.command(name="channelinfo", description="Details about a text channel")
@app_commands.describe(channel="Channel (default: this one)")
async def info_channelinfo(interaction: discord.Interaction, channel: Optional[discord.TextChannel] = None):
    ch = channel or interaction.channel
    e = base_embed(title=f"💬 #{ch.name}")
    e.add_field(name="🆔 ID", value=str(ch.id), inline=True)
    e.add_field(name="📁 Category", value=ch.category.name if ch.category else "None", inline=True)
    e.add_field(name="🐢 Slowmode", value=f"{ch.slowmode_delay}s", inline=True)
    e.add_field(name="🔞 NSFW", value="Yes" if ch.is_nsfw() else "No", inline=True)
    e.add_field(name="📅 Created", value=discord.utils.format_dt(ch.created_at, "D"), inline=True)
    e.add_field(name="📝 Topic", value=(ch.topic or "None")[:1000], inline=False)
    await interaction.response.send_message(embed=e)


@info.command(name="membercount", description="Humans, bots and total members")
async def info_membercount(interaction: discord.Interaction):
    g = interaction.guild
    bots_n = sum(1 for m in g.members if m.bot)
    await interaction.response.send_message(embed=info_embed("👥 Member Count", f"**Total:** {g.member_count}\n**Humans:** {g.member_count - bots_n}\n**Bots:** {bots_n}"))


@info.command(name="servericon", description="Show the server icon")
async def info_servericon(interaction: discord.Interaction):
    if not interaction.guild.icon:
        return await interaction.response.send_message("This server has no icon.", ephemeral=True)
    e = base_embed(title=f"🖼️ {interaction.guild.name}")
    e.set_image(url=interaction.guild.icon.url)
    await interaction.response.send_message(embed=e)


@info.command(name="roles", description="List the server's roles")
async def info_roles(interaction: discord.Interaction):
    roles = [r.mention for r in reversed(interaction.guild.roles) if not r.is_default()]
    await interaction.response.send_message(embed=info_embed(f"🎭 Roles ({len(roles)})", (" ".join(roles) or "No roles")[:4000]),
                                            allowed_mentions=discord.AllowedMentions.none())


@info.command(name="emojis", description="List the server's custom emojis")
async def info_emojis(interaction: discord.Interaction):
    em = [str(e) for e in interaction.guild.emojis]
    await interaction.response.send_message(embed=info_embed(f"😀 Emojis ({len(em)})", (" ".join(em) or "No custom emojis")[:4000]))


@info.command(name="boosts", description="Server boost status")
async def info_boosts(interaction: discord.Interaction):
    g = interaction.guild
    await interaction.response.send_message(embed=info_embed("🚀 Boosts", f"**Level:** {g.premium_tier}\n**Boosts:** {g.premium_subscription_count}\n**Boosters:** {len(g.premium_subscribers)}"))


@info.command(name="joined", description="When a member joined and their join position")
@app_commands.describe(member="Member (default: you)")
async def info_joined(interaction: discord.Interaction, member: Optional[discord.Member] = None):
    member = member or interaction.user
    ordered = sorted((m for m in interaction.guild.members if m.joined_at), key=lambda m: m.joined_at)
    pos = next((i + 1 for i, m in enumerate(ordered) if m.id == member.id), None)
    when = discord.utils.format_dt(member.joined_at, "F") if member.joined_at else "unknown"
    await interaction.response.send_message(embed=info_embed(f"📥 {member.display_name}", f"**Joined:** {when}\n**Join position:** #{pos or '?'}"))


@info.command(name="accountage", description="How old a Discord account is")
@app_commands.describe(member="Member (default: you)")
async def info_accountage(interaction: discord.Interaction, member: Optional[discord.Member] = None):
    member = member or interaction.user
    days = (discord.utils.utcnow() - member.created_at).days
    await interaction.response.send_message(embed=info_embed(f"📅 {member.display_name}", f"Created {discord.utils.format_dt(member.created_at, 'F')}\nThat's **{days:,} days** ago."))


@info.command(name="permissions", description="Key permissions a member has in this channel")
@app_commands.describe(member="Member (default: you)")
async def info_permissions(interaction: discord.Interaction, member: Optional[discord.Member] = None):
    member = member or interaction.user
    p = interaction.channel.permissions_for(member)
    key = ["administrator", "manage_guild", "manage_roles", "manage_channels", "manage_messages", "kick_members", "ban_members",
           "moderate_members", "mention_everyone", "attach_files", "embed_links", "send_messages", "view_channel"]
    have = [k.replace("_", " ") for k in key if getattr(p, k)]
    await interaction.response.send_message(embed=info_embed(f"🔑 {member.display_name} here", ", ".join(have) or "No key permissions here."))


@info.command(name="inrole", description="List members who have a role")
@app_commands.describe(role="The role")
async def info_inrole(interaction: discord.Interaction, role: discord.Role):
    names = [m.mention for m in role.members][:40]
    await interaction.response.send_message(embed=info_embed(f"🎭 {role.name}: {len(role.members)} member(s)", " ".join(names) or "Nobody has this role."),
                                            allowed_mentions=discord.AllowedMentions.none())


@info.command(name="bots", description="List the bots in this server")
async def info_bots(interaction: discord.Interaction):
    names = [m.mention for m in interaction.guild.members if m.bot]
    await interaction.response.send_message(embed=info_embed(f"🤖 Bots ({len(names)})", " ".join(names) or "No bots."),
                                            allowed_mentions=discord.AllowedMentions.none())


# ─── /tools ───────────────────────────────────────────────────────────────

tools = make_group("tools", "Handy utilities: calculator, polls, reminders and more")

_OPS = {ast.Add: operator.add, ast.Sub: operator.sub, ast.Mult: operator.mul, ast.Div: operator.truediv, ast.Pow: operator.pow,
        ast.Mod: operator.mod, ast.FloorDiv: operator.floordiv, ast.USub: operator.neg, ast.UAdd: operator.pos}


def safe_eval(expr: str):
    def ev(n):
        if isinstance(n, ast.Expression):
            return ev(n.body)
        if isinstance(n, ast.Constant) and isinstance(n.value, (int, float)) and not isinstance(n.value, bool):
            return n.value
        if isinstance(n, ast.BinOp) and type(n.op) in _OPS:
            left, right = ev(n.left), ev(n.right)
            if isinstance(n.op, ast.Pow) and (abs(right) > 100 or abs(left) > 1e6):
                raise ValueError("exponent too large")
            return _OPS[type(n.op)](left, right)
        if isinstance(n, ast.UnaryOp) and type(n.op) in _OPS:
            return _OPS[type(n.op)](ev(n.operand))
        raise ValueError("unsupported expression")
    return ev(ast.parse(expr.strip(), mode="eval"))


@tools.command(name="calc", description="Calculate a math expression like (12+8)*3/4")
@app_commands.describe(expression="Numbers and + - * / ** % // ( )")
async def tools_calc(interaction: discord.Interaction, expression: app_commands.Range[str, 1, 200]):
    try:
        result = safe_eval(expression)
    except ZeroDivisionError:
        return await interaction.response.send_message("🙏 Division by zero isn't possible.", ephemeral=True)
    except (ValueError, SyntaxError, OverflowError, TypeError):
        return await interaction.response.send_message("🙏 I can only do numbers with + - * / ** % // and brackets.", ephemeral=True)
    await interaction.response.send_message(embed=info_embed("🧮 Calculator", f"`{expression}` = **{result:,}**" if isinstance(result, int) else f"`{expression}` = **{result:,.10g}**"))


@tools.command(name="dice", description="Roll one or more dice")
@app_commands.describe(sides="Sides per die (default 6)", count="How many dice (default 1)")
async def tools_dice(interaction: discord.Interaction, sides: app_commands.Range[int, 2, 1000] = 6, count: app_commands.Range[int, 1, 20] = 1):
    rolls = [random.randint(1, sides) for _ in range(count)]
    await interaction.response.send_message(embed=info_embed("🎲 Dice", f"{count}d{sides}: {', '.join(map(str, rolls))}\n**Total: {sum(rolls)}**"))


@tools.command(name="coinflip", description="Flip a coin")
async def tools_coinflip(interaction: discord.Interaction):
    await interaction.response.send_message(embed=info_embed("🪙 Coin Flip", f"**{random.choice(['Heads', 'Tails'])}**"))


@tools.command(name="choose", description="Pick one option at random")
@app_commands.describe(options="Options separated by commas or |")
async def tools_choose(interaction: discord.Interaction, options: app_commands.Range[str, 3, 500]):
    items = [o.strip() for o in re.split(r"[,|]", options) if o.strip()]
    if len(items) < 2:
        return await interaction.response.send_message("Give me at least two options separated by commas.", ephemeral=True)
    await interaction.response.send_message(embed=info_embed("🎯 I choose…", f"**{random.choice(items)}**"))


@tools.command(name="poll", description="Create a quick poll with up to 4 options")
@app_commands.describe(question="The question", option1="First option", option2="Second option", option3="Third option", option4="Fourth option")
async def tools_poll(interaction: discord.Interaction, question: app_commands.Range[str, 1, 250], option1: app_commands.Range[str, 1, 100],
                     option2: app_commands.Range[str, 1, 100], option3: Optional[str] = None, option4: Optional[str] = None):
    opts = [o for o in (option1, option2, option3, option4) if o]
    emojis = ["1️⃣", "2️⃣", "3️⃣", "4️⃣"]
    e = base_embed(title=f"📊 {question}", description="\n".join(f"{emojis[i]} {o[:100]}" for i, o in enumerate(opts)))
    e.set_footer(text=f"Poll by {interaction.user.display_name}")
    await interaction.response.send_message(embed=e)
    msg = await interaction.original_response()
    for i in range(len(opts)):
        await msg.add_reaction(emojis[i])


async def _remind(user_id: int, channel_id: int, minutes: int, text: str):
    await asyncio.sleep(minutes * 60)
    ch = bot.get_channel(channel_id)
    if ch:
        await ch.send(f"⏰ <@{user_id}> reminder: {text}", allowed_mentions=discord.AllowedMentions(users=[discord.Object(id=user_id)]))


@tools.command(name="remind", description="Get a reminder in this channel after some minutes")
@app_commands.describe(minutes="Minutes from now (1-10080)", text="What to remind you about")
async def tools_remind(interaction: discord.Interaction, minutes: app_commands.Range[int, 1, 10080], text: app_commands.Range[str, 1, 300]):
    task = asyncio.create_task(_remind(interaction.user.id, interaction.channel_id, minutes, text))
    bg_tasks.add(task)
    task.add_done_callback(bg_tasks.discard)
    await interaction.response.send_message(embed=info_embed("⏰ Reminder Set", f"I'll remind you in **{minutes} minute(s)**.\n*(Reminders reset if I restart.)*"), ephemeral=True)


@tools.command(name="timestamp", description="Make Discord timestamps from an IST date and time")
@app_commands.describe(date_time="Format: YYYY-MM-DD HH:MM (24-hour, Indian time)")
async def tools_timestamp(interaction: discord.Interaction, date_time: str):
    try:
        dt = datetime.datetime.strptime(date_time.strip(), "%Y-%m-%d %H:%M").replace(tzinfo=IST)
    except ValueError:
        return await interaction.response.send_message("Use the format `YYYY-MM-DD HH:MM`, for example `2026-11-08 18:30`.", ephemeral=True)
    ts = int(dt.timestamp())
    rows = [f"`<t:{ts}:{s}>` → <t:{ts}:{s}>" for s in ("F", "f", "D", "t", "R")]
    await interaction.response.send_message(embed=info_embed("🕒 Timestamps (IST input)", "\n".join(rows)))


@tools.command(name="announce", description="Post an announcement embed in a channel")
@app_commands.describe(channel="Where to post", message="The announcement text")
@app_commands.checks.has_permissions(manage_guild=True)
async def tools_announce(interaction: discord.Interaction, channel: discord.TextChannel, message: app_commands.Range[str, 1, 3500]):
    await channel.send(embed=info_embed("📢 Announcement", message))
    await interaction.response.send_message(f"✅ Posted in {channel.mention}.", ephemeral=True)


@tools.command(name="embed", description="Post a custom embed with a title and text")
@app_commands.describe(title="Embed title", description="Embed text", channel="Where to post (default: here)")
@app_commands.checks.has_permissions(manage_messages=True)
async def tools_embed(interaction: discord.Interaction, title: app_commands.Range[str, 1, 250], description: app_commands.Range[str, 1, 3500],
                      channel: Optional[discord.TextChannel] = None):
    target = channel or interaction.channel
    await target.send(embed=info_embed(title, description))
    await interaction.response.send_message(f"✅ Posted in {target.mention}.", ephemeral=True)


@tools.command(name="wordcount", description="Count words and characters in some text")
@app_commands.describe(text="Your text")
async def tools_wordcount(interaction: discord.Interaction, text: app_commands.Range[str, 1, 4000]):
    await interaction.response.send_message(embed=info_embed("📝 Word Count", f"**Words:** {len(text.split())}\n**Characters:** {len(text)}"))


@tools.command(name="countdown", description="Days remaining until a date")
@app_commands.describe(date="Format: YYYY-MM-DD")
async def tools_countdown(interaction: discord.Interaction, date: str):
    try:
        target = datetime.datetime.strptime(date.strip(), "%Y-%m-%d").date()
    except ValueError:
        return await interaction.response.send_message("Use the format `YYYY-MM-DD`.", ephemeral=True)
    days = (target - ist_now().date()).days
    text = f"**{days}** day(s) to go" if days > 0 else ("**Today!** 🎉" if days == 0 else f"That was **{-days}** day(s) ago")
    await interaction.response.send_message(embed=info_embed(f"⏳ Countdown to {target:%d %b %Y}", text))


@tools.command(name="randomnumber", description="A random number between two values")
@app_commands.describe(minimum="Lowest value", maximum="Highest value")
async def tools_randomnumber(interaction: discord.Interaction, minimum: int, maximum: int):
    low, high = min(minimum, maximum), max(minimum, maximum)
    await interaction.response.send_message(embed=info_embed("🔢 Random Number", f"**{random.randint(low, high)}** (between {low} and {high})"))


@tools.command(name="snowflake", description="Find when a Discord ID was created")
@app_commands.describe(id="Any Discord user, channel, message or server ID")
async def tools_snowflake(interaction: discord.Interaction, id: str):
    if not id.strip().isdigit():
        return await interaction.response.send_message("Give me a numeric Discord ID.", ephemeral=True)
    when = discord.utils.snowflake_time(int(id))
    await interaction.response.send_message(embed=info_embed("🆔 Snowflake", f"Created {discord.utils.format_dt(when, 'F')} ({discord.utils.format_dt(when, 'R')})"))


@tools.command(name="pickwinner", description="Pick a random member (optionally from one role)")
@app_commands.describe(role="Only pick from this role")
async def tools_pickwinner(interaction: discord.Interaction, role: Optional[discord.Role] = None):
    pool = [m for m in (role.members if role else interaction.guild.members) if not m.bot]
    if not pool:
        return await interaction.response.send_message("Nobody to pick from.", ephemeral=True)
    await interaction.response.send_message(embed=info_embed("🎉 Winner!", f"{random.choice(pool).mention}"), allowed_mentions=discord.AllowedMentions.none())


# ─── /ticket ──────────────────────────────────────────────────────────────

ticket = make_group("ticket", "Manage verification tickets")


@ticket.command(name="panel", description="Post the verification ticket panel")
@app_commands.checks.has_permissions(manage_guild=True)
async def ticket_panel(interaction: discord.Interaction):
    ch = await post_ticket_panel(interaction.guild)
    await interaction.response.send_message(f"✅ Panel posted in {ch.mention}." if ch else "⚠️ No verify channel set. Use `/config setverifychannel`.", ephemeral=True)


def _in_ticket(interaction: discord.Interaction) -> bool:
    return ticket_owner_id(interaction.channel) is not None


@ticket.command(name="close", description="Close this ticket")
async def ticket_close(interaction: discord.Interaction):
    ch = interaction.channel
    owner = ticket_owner_id(ch)
    if owner is None:
        return await deny(interaction, "This isn't a ticket channel.")
    if interaction.user.id != owner and not ch.permissions_for(interaction.user).manage_channels:
        return await deny(interaction, "Only the ticket owner or staff can close it.")
    await interaction.response.send_message("Closing this ticket in 5 seconds... 🙏")
    await log_event(interaction.guild, base_embed(title="🔒 Ticket Closed", description=f"`{ch.name}` closed by {interaction.user.mention}"))
    await asyncio.sleep(5)
    try:
        await ch.delete(reason=f"Ticket closed by {interaction.user}")
    except (discord.Forbidden, discord.NotFound):
        pass


@ticket.command(name="add", description="Add a member to this ticket")
@app_commands.describe(member="Member to add")
@app_commands.checks.has_permissions(manage_channels=True)
async def ticket_add(interaction: discord.Interaction, member: discord.Member):
    if not _in_ticket(interaction):
        return await deny(interaction, "This isn't a ticket channel.")
    await interaction.channel.set_permissions(member, view_channel=True, send_messages=True, read_message_history=True)
    await interaction.response.send_message(embed=info_embed("➕ Added", f"{member.mention} can now see this ticket."))


@ticket.command(name="remove", description="Remove a member from this ticket")
@app_commands.describe(member="Member to remove")
@app_commands.checks.has_permissions(manage_channels=True)
async def ticket_remove(interaction: discord.Interaction, member: discord.Member):
    if not _in_ticket(interaction):
        return await deny(interaction, "This isn't a ticket channel.")
    if member.id == ticket_owner_id(interaction.channel):
        return await deny(interaction, "You can't remove the ticket owner.")
    await interaction.channel.set_permissions(member, overwrite=None)
    await interaction.response.send_message(embed=info_embed("➖ Removed", f"{member.mention} can no longer see this ticket."))


@ticket.command(name="rename", description="Rename this ticket channel")
@app_commands.describe(name="New channel name")
@app_commands.checks.has_permissions(manage_channels=True)
async def ticket_rename(interaction: discord.Interaction, name: app_commands.Range[str, 1, 80]):
    if not _in_ticket(interaction):
        return await deny(interaction, "This isn't a ticket channel.")
    await interaction.channel.edit(name=safe_channel_name(name))
    await interaction.response.send_message(embed=info_embed("✏️ Renamed", f"This ticket is now `{safe_channel_name(name)}`."))


@ticket.command(name="claim", description="Claim this ticket as the staff member handling it")
@app_commands.checks.has_permissions(manage_messages=True)
async def ticket_claim(interaction: discord.Interaction):
    if not _in_ticket(interaction):
        return await deny(interaction, "This isn't a ticket channel.")
    await interaction.response.send_message(embed=info_embed("🙋 Ticket Claimed", f"{interaction.user.mention} will take care of this verification."))


# ─── Top-level commands ───────────────────────────────────────────────────


@bot.tree.command(name="sendmessage", description="Post the verification ticket panel in the verify channel")
@app_commands.checks.has_permissions(manage_guild=True)
async def sendmessage(interaction: discord.Interaction):
    ch = await post_ticket_panel(interaction.guild)
    if ch:
        await interaction.response.send_message(f"✅ Verification panel posted in {ch.mention}.", ephemeral=True)
    else:
        await interaction.response.send_message("⚠️ No verify channel is set. Use `/config setverifychannel` first.", ephemeral=True)


@bot.tree.command(name="help", description="Show every command, grouped")
async def help_cmd(interaction: discord.Interaction):
    e = base_embed(
        title="📖 Command Scriptures",
        description=(f"**{total_commands()} slash commands.** Type `/` and pick a group, for example `/ai ask`, `/dharma verse` or `/level rank`.\n"
                     "You can also just **@mention me or reply to my messages** and chat like with a friend."))
    for cmd in sorted(bot.tree.get_commands(), key=lambda c: c.name):
        if isinstance(cmd, app_commands.Group):
            names = " · ".join(f"`{c.name}`" for c in cmd.commands)
            e.add_field(name=f"/{cmd.name} ({len(cmd.commands)})", value=f"{cmd.description}\n{names}"[:1020], inline=False)
    e.set_footer(text="Hari Om 🙏 | /sendmessage posts the verification panel (admins)")
    await interaction.response.send_message(embed=e, ephemeral=True)


# ═══════════════════════════════════════════════════════════════════════════
# Entrypoint
# ═══════════════════════════════════════════════════════════════════════════

if __name__ == "__main__":
    if not DISCORD_TOKEN or "PUT_YOUR" in DISCORD_TOKEN:
        raise SystemExit("❌ Put your Discord bot token into DISCORD_TOKEN in the CONFIG section at the top of this file.")
    if not ai_ready():
        print("⚠️  GROQ_API_KEY isn't set: AI chat, /ai commands and /ai deepsearch summaries won't work until you add the free key.")
    bot.run(DISCORD_TOKEN)
