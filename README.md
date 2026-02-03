# Ai-chatbot-2
#!/usr/bin/env python3
"""
Super Legend Chatbot - Single-file CLI (no external packages required)

Copy and paste this into a file (e.g., super_legend_cli.py) and run:
    python3 super_legend_cli.py

This version uses only Python's standard library so it will run on environments
without Flask, numpy, or scikit-learn (for example Pydroid3 without extra installs).
It provides:
- Category-aware replies (jokes, greetings, stocks placeholder, business ideas, dropship)
- Simple rule-based intent detection
- Session memory (in-memory during run; optional file save)
- Emoji-rich responses
- A small interactive REPL and a "batch" mode via command-line argument
"""

import sys
import time
import json
import random
import re
import os

# -------------------------
# Simple rule-based intent detector
# -------------------------
INTENT_KEYWORDS = {
    'greeting': ['hello', 'hi', 'hey', 'good morning', 'good evening', 'greetings'],
    'joke': ['joke', 'make me laugh', 'funny', 'tell me a joke'],
    'stock': ['stock', 'ticker', 'price', 'aapl', 'tsla', 'goog', 'msft'],
    'business_idea': ['business idea', 'startup idea', 'idea', 'business'],
    'dropship': ['dropship', 'dropshipping', 'dropshipper', 'how to dropship'],
    'general': ['what is', 'who is', 'explain', 'how to', 'define']
}

def detect_intent(text: str) -> str:
    t = text.lower()
    # direct exact matches first
    for intent, kws in INTENT_KEYWORDS.items():
        for kw in kws:
            if kw in t:
                return intent
    # fallback heuristics
    if re.search(r'\b[A-Z]{1,5}\b', text) and any(ch.isupper() for ch in text):
        return 'stock'
    if len(text.split()) <= 2 and text.strip().endswith('?'):
        return 'general'
    return 'fallback'

# -------------------------
# Memory (in-memory with optional file persistence)
# -------------------------
class Memory:
    def __init__(self, persist_file=None):
        self.persist_file = persist_file
        self.sessions = {}  # user_id -> list of (role, text, ts)
        if persist_file and os.path.exists(persist_file):
            try:
                with open(persist_file, 'r', encoding='utf-8') as f:
                    self.sessions = json.load(f)
            except Exception:
                self.sessions = {}

    def append(self, user_id, role, text):
        ts = int(time.time())
        self.sessions.setdefault(user_id, []).append({'role': role, 'text': text, 'ts': ts})
        if self.persist_file:
            try:
                with open(self.persist_file, 'w', encoding='utf-8') as f:
                    json.dump(self.sessions, f, ensure_ascii=False, indent=2)
            except Exception:
                pass

    def get(self, user_id, limit=20):
        return self.sessions.get(user_id, [])[-limit:]

# -------------------------
# Response generator (templates + small dynamic behavior)
# -------------------------
TEMPLATES = {
    'greeting': [
        "Hey there! 👋 How can I help you today? 😊",
        "Hello! Ready to chat — what's up? 🤖✨",
        "Hi! I'm Super Legend — pick a category: jokes, stocks, business ideas, dropship. 🚀"
    ],
    'joke': [
        "Why did the developer go broke? Because he used up all his cache. 😅",
        "I told my computer I needed a break — it said 'No problem, I'll go to sleep.' 😴",
        "Why do programmers prefer dark mode? Because light attracts bugs. 🐛😆"
    ],
    'stock': [
        "I can fetch live stock prices if you enable remote fetch. Tell me the ticker (e.g., AAPL). 📈",
        "Send me a ticker symbol and I'll check the latest price for you. 💹"
    ],
    'business_idea': [
        "Try a niche dropship store for eco-friendly pet products — low competition, high margins. 🐶🌿",
        "Subscription boxes for local snacks with influencer marketing. 📦🔥",
        "Microservices for local businesses: setup, automation, and monthly maintenance. 🛠️💼"
    ],
    'dropship': [
        "Dropshipping checklist: niche research; supplier vetting; fast shipping; clear returns. ✅",
        "Use product bundles and social proof to increase AOV. 💡",
        "Test ads with small budgets and scale winners. Track shipping times closely. ⏱️"
    ],
    'general': [
        "Here's a quick answer: {answer} 🔎",
        "Good question — {answer} ✅"
    ],
    'fallback': [
        "Sorry, I didn't get that — want a joke or a business idea? 😅",
        "I'm not sure I understood. Try 'joke', 'stock AAPL', or 'give me a business idea'. ✨"
    ]
}

def knowledge_lookup(text: str) -> str:
    t = text.lower()
    if 'photosynthesis' in t:
        return "Photosynthesis converts light into chemical energy in plants."
    if 'ai' in t and 'best' in t:
        return "Choose models based on latency, cost, and privacy needs."
    if 'python' in t and 'how' in t:
        return "Start with official docs and small projects; practice daily."
    return "I don't have that locally; I can look it up if you allow remote access."

def generate_response(text: str, intent: str, user_id: str, memory: Memory) -> str:
    # Keyword overrides for better UX
    lowered = text.lower()
    if any(k in lowered for k in ['joke', 'tell me a joke', 'make me laugh']):
        intent = 'joke'
    if any(k in lowered for k in ['hello', 'hi', 'hey']):
        intent = 'greeting'
    if any(k in lowered for k in ['dropship', 'dropshipping', 'how to dropship']):
        intent = 'dropship'
    if any(k in lowered for k in ['business idea', 'startup idea', 'give me a business idea']):
        intent = 'business_idea'
    if any(k in lowered for k in ['stock', 'price', 'ticker', 'aapl', 'tsla']):
        intent = 'stock'

    # Template reply
    if intent in TEMPLATES and random.random() > 0.05:
        reply = random.choice(TEMPLATES[intent])
        if '{answer}' in reply:
            reply = reply.format(answer=knowledge_lookup(text))
        return reply

    # Stock handling
    if intent == 'stock':
        m = re.search(r'\b([A-Z]{1,5})\b', text)
        if m:
            ticker = m.group(1)
            return f"I can check {ticker} for you — enable remote fetch to get live data. 📊"
        return "Tell me the stock ticker symbol (e.g., TSLA, AAPL). 📈"

    # General knowledge
    if intent == 'general':
        return knowledge_lookup(text)

    # Fallback
    return random.choice(TEMPLATES['fallback'])

# -------------------------
# CLI / REPL
# -------------------------
def repl(user_id='you', persist_file=None):
    mem = Memory(persist_file=persist_file)
    print("Super Legend Chatbot (CLI) — type 'exit' or 'quit' to stop.")
    print("Try: 'Tell me a joke', 'Stock AAPL', 'Give me a business idea', 'How to dropship'")
    while True:
        try:
            text = input("\nYou: ").strip()
        except (EOFError, KeyboardInterrupt):
            print("\nGoodbye.")
            break
        if not text:
            continue
        if text.lower() in ('exit', 'quit', 'bye'):
            print("Bot: Bye! 👋")
            break
        intent = detect_intent(text)
        resp = generate_response(text, intent, user_id, mem)
        mem.append(user_id, 'user', text)
        mem.append(user_id, 'bot', resp)
        print("Bot:", resp)

# -------------------------
# Batch mode (single-shot) for quick testing
# -------------------------
def single_query(text, user_id='you', persist_file=None):
    mem = Memory(persist_file=persist_file)
    intent = detect_intent(text)
    resp = generate_response(text, intent, user_id, mem)
    mem.append(user_id, 'user', text)
    mem.append(user_id, 'bot', resp)
    return resp, intent

# -------------------------
# Entry point
# -------------------------
def print_help():
    print("Usage:")
    print("  python3 super_legend_cli.py           # start interactive chat")
    print("  python3 super_legend_cli.py --say 'Your question'   # single query mode")
    print("  python3 super_legend_cli.py --persist file.json     # save session to file")

if __name__ == '__main__':
    args = sys.argv[1:]
    if not args:
        repl(user_id='you', persist_file=None)
    else:
        # parse simple args
        if '--help' in args or '-h' in args:
            print_help()
            sys.exit(0)
        if '--say' in args:
            try:
                idx = args.index('--say')
                q = args[idx + 1]
            except Exception:
                print("Error: provide a query after --say")
                sys.exit(1)
            resp, intent = single_query(q)
            print("Intent:", intent)
            print("Bot:", resp)
            sys.exit(0)
        if '--persist' in args:
            try:
                idx = args.index('--persist')
                pf = args[idx + 1]
            except Exception:
                print("Error: provide a filename after --persist")
                sys.exit(1)
            repl(user_id='you', persist_file=pf)
        else:
            # treat all args as a single message
            text = " ".join(args)
            resp, intent = single_query(text)
            print("Intent:", intent)
            print("Bot:", resp)


RESULT‼️‼️:-
Super Legend Chatbot (CLI) — type 'exit' or 'quit' to stop.
Try: 'Tell me a joke', 'Stock AAPL', 'Give me a business idea', 'How to dropship'

You: hi
Bot: Hi! I'm Super Legend — pick a category: jokes, stocks, business ideas, dropship. 🚀

You: what about stock appl
Bot: I can fetch live stock prices if you enable remote fetch. Tell me the ticker (e.g., AAPL). 📈

You: ok give me a bussines idea
Bot: Microservices for local businesses: setup, automation, and monthly maintenance. 🛠️💼

You: dtopship
Bot: Hello! Ready to chat — what's up? 🤖✨

You: hi
Bot: Hey there! 👋 How can I help you today? 😊

You: hello
Bot: I'm not sure I understood. Try 'joke', 'stock AAPL', or 'give me a business idea'. ✨

You: how ro dropship
Bot: Dropshipping checklist: niche research; supplier vetting; fast shipping; clear returns. ✅

You: joke
Bot: Why did the developer go broke? Because he used up all his cache. 😅

You: joke
Bot: Why do programmers prefer dark mode? Because light attracts bugs. 🐛😆

You: joke
Bot: Why do programmers prefer dark mode? Because light attracts bugs. 🐛😆

You: business idea
Bot: Subscription boxes for local snacks with influencer marketing. 📦🔥

You: bussines idea
Bot: Try a niche dropship store for eco-friendly pet products — low competition, high margins. 🐶🌿

You: hey
Bot: Hello! Ready to chat — what's up? 🤖✨

You: ok.bye
Bot: Sorry, I didn't get that — want a joke or a business idea? 😅

You: bye
Bot: Bye! 👋

[Program finished]
