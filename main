“””
EscrowBot - Global Escrow & Micro-Trade Telegram Bot
Single-file architecture: FastAPI + aiogram + SQLite + React (CDN, in-browser Babel)
Deploy on Render: start command -> uvicorn main:app –host 0.0.0.0 –port $PORT
“””

import asyncio
import hashlib
import hmac
import json
import logging
import os
import uuid
import enum
from contextlib import asynccontextmanager
from datetime import datetime, timedelta, timezone
from typing import Optional
from urllib.parse import parse_qsl, unquote

from dotenv import load_dotenv
load_dotenv()

from fastapi import Depends, FastAPI, HTTPException, Header, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import HTMLResponse, JSONResponse
from pydantic import BaseModel, Field
from sqlalchemy import (
BigInteger, Boolean, DateTime, Enum as SAEnum,
Float, ForeignKey, String, Text, select, update, func
)
from sqlalchemy.ext.asyncio import (
AsyncSession, async_sessionmaker, create_async_engine
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from aiogram import Bot, Dispatcher, Router
from aiogram.filters import CommandStart, Command
from aiogram.types import (
InlineKeyboardButton, InlineKeyboardMarkup, Message, Update, WebAppInfo
)

# —————————————————————————

# Config

# —————————————————————————

BOT_TOKEN    = os.getenv(“BOT_TOKEN”, “”)
SECRET_KEY   = os.getenv(“SECRET_KEY”, “changeme-32chars-minimum-secret!”)
WEBHOOK_URL  = os.getenv(“WEBHOOK_URL”, “”)
MINI_APP_URL = os.getenv(“MINI_APP_URL”, “”)
ENVIRONMENT  = os.getenv(“ENVIRONMENT”, “development”)
DATABASE_URL = os.getenv(“DATABASE_URL”, “sqlite+aiosqlite:///./escrow.db”)
BOT_USERNAME = os.getenv(“BOT_USERNAME”, “EscrowBot”)

FEE_PERCENT   = 1.5
FEE_FLAT      = 1.0
FEE_THRESHOLD = 50.0

logging.basicConfig(level=logging.INFO)
log = logging.getLogger(**name**)

def calculate_fee(amount: float) -> tuple[float, str]:
if amount >= FEE_THRESHOLD:
return round(amount * FEE_PERCENT / 100, 2), “percentage”
return FEE_FLAT, “flat”

# —————————————————————————

# Database

# —————————————————————————

engine = create_async_engine(DATABASE_URL, echo=False)
AsyncSessionLocal = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

class Base(DeclarativeBase):
pass

class DealStatus(str, enum.Enum):
PENDING   = “pending”
FUNDED    = “funded”
DELIVERED = “delivered”
COMPLETED = “completed”
DISPUTED  = “disputed”
CANCELLED = “cancelled”

class User(Base):
**tablename** = “users”
id:               Mapped[int]         = mapped_column(BigInteger, primary_key=True, autoincrement=True)
telegram_id:      Mapped[int]         = mapped_column(BigInteger, unique=True, index=True)
username:         Mapped[str | None]  = mapped_column(String(64), nullable=True)
first_name:       Mapped[str]         = mapped_column(String(128), default=“User”)
language_code:    Mapped[str]         = mapped_column(String(8), default=“en”)
wallet_balance:   Mapped[float]       = mapped_column(Float, default=0.0)
total_trades:     Mapped[int]         = mapped_column(default=0)
reputation_score: Mapped[float]       = mapped_column(Float, default=5.0)
created_at:       Mapped[datetime]    = mapped_column(DateTime(timezone=True), server_default=func.now())
deals_as_seller = relationship(“Deal”, foreign_keys=“Deal.seller_id”, back_populates=“seller”, lazy=“selectin”)
deals_as_buyer  = relationship(“Deal”, foreign_keys=“Deal.buyer_id”,  back_populates=“buyer”,  lazy=“selectin”)

class Deal(Base):
**tablename** = “deals”
id:             Mapped[int]           = mapped_column(BigInteger, primary_key=True, autoincrement=True)
deal_uid:       Mapped[str]           = mapped_column(String(36), unique=True, index=True, default=lambda: str(uuid.uuid4()))
seller_id:      Mapped[int]           = mapped_column(BigInteger, ForeignKey(“users.telegram_id”))
buyer_id:       Mapped[int | None]    = mapped_column(BigInteger, ForeignKey(“users.telegram_id”), nullable=True)
title:          Mapped[str]           = mapped_column(String(256))
description:    Mapped[str | None]    = mapped_column(Text, nullable=True)
amount:         Mapped[float]         = mapped_column(Float)
fee:            Mapped[float]         = mapped_column(Float, default=0.0)
currency:       Mapped[str]           = mapped_column(String(8), default=“USD”)
status:         Mapped[DealStatus]    = mapped_column(SAEnum(DealStatus, name=“deal_status”), default=DealStatus.PENDING)
dispute_reason: Mapped[str | None]    = mapped_column(Text, nullable=True)
delivery_proof: Mapped[str | None]    = mapped_column(Text, nullable=True)
funded_at:      Mapped[datetime|None] = mapped_column(DateTime(timezone=True), nullable=True)
delivered_at:   Mapped[datetime|None] = mapped_column(DateTime(timezone=True), nullable=True)
completed_at:   Mapped[datetime|None] = mapped_column(DateTime(timezone=True), nullable=True)
created_at:     Mapped[datetime]      = mapped_column(DateTime(timezone=True), server_default=func.now())
seller = relationship(“User”, foreign_keys=[seller_id], back_populates=“deals_as_seller”)
buyer  = relationship(“User”, foreign_keys=[buyer_id],  back_populates=“deals_as_buyer”)

async def get_db():
async with AsyncSessionLocal() as session:
try:
yield session
except Exception:
await session.rollback()
raise

async def create_tables():
async with engine.begin() as conn:
await conn.run_sync(Base.metadata.create_all)

# —————————————————————————

# Auth helper

# —————————————————————————

def verify_telegram_init_data(init_data: str) -> dict | None:
try:
parsed = dict(parse_qsl(init_data, keep_blank_values=True))
hash_value = parsed.pop(“hash”, None)
if not hash_value:
return None
data_check = “\n”.join(f”{k}={v}” for k, v in sorted(parsed.items()))
secret = hmac.new(b”WebAppData”, BOT_TOKEN.encode(), hashlib.sha256).digest()
expected = hmac.new(secret, data_check.encode(), hashlib.sha256).hexdigest()
if not hmac.compare_digest(expected, hash_value):
return None
user_raw = parsed.get(“user”)
return json.loads(unquote(user_raw)) if user_raw else parsed
except Exception:
return None

async def get_or_create_user(db: AsyncSession, tg_id: int, data: dict) -> User:
res = await db.execute(select(User).where(User.telegram_id == tg_id))
user = res.scalar_one_or_none()
if not user:
user = User(
telegram_id=tg_id,
username=data.get(“username”),
first_name=data.get(“first_name”, “User”),
language_code=data.get(“language_code”, “en”),
)
db.add(user)
await db.commit()
await db.refresh(user)
return user

def deal_to_dict(deal: Deal) -> dict:
return {
“id”:             deal.id,
“deal_uid”:       deal.deal_uid,
“seller_id”:      deal.seller_id,
“buyer_id”:       deal.buyer_id,
“title”:          deal.title,
“description”:    deal.description,
“amount”:         deal.amount,
“fee”:            deal.fee,
“currency”:       deal.currency,
“status”:         deal.status.value,
“dispute_reason”: deal.dispute_reason,
“delivery_proof”: deal.delivery_proof,
“funded_at”:      deal.funded_at.isoformat()    if deal.funded_at    else None,
“delivered_at”:   deal.delivered_at.isoformat() if deal.delivered_at else None,
“completed_at”:   deal.completed_at.isoformat() if deal.completed_at else None,
“created_at”:     deal.created_at.isoformat()   if deal.created_at   else None,
}

# —————————————————————————

# Pydantic schemas

# —————————————————————————

class DealCreate(BaseModel):
title:       str   = Field(…, min_length=3, max_length=256)
description: Optional[str] = Field(None, max_length=2000)
amount:      float = Field(…, gt=0, le=100000)
currency:    str   = Field(default=“USD”, max_length=8)

class FundBody(BaseModel):
payment_reference: Optional[str] = None

class DeliverBody(BaseModel):
proof: str = Field(…, min_length=5, max_length=2000)

class DisputeBody(BaseModel):
reason: str = Field(…, min_length=10, max_length=1000)

# —————————————————————————

# Telegram Bot

# —————————————————————————

bot_router = Router()

@bot_router.message(CommandStart())
async def cmd_start(msg: Message):
url = MINI_APP_URL or “https://t.me”
args = (msg.text or “”).split(” “, 1)
deep = args[1] if len(args) > 1 else None
if deep and deep.startswith(“deal_”):
uid = deep[5:]
kb = InlineKeyboardMarkup(inline_keyboard=[[
InlineKeyboardButton(text=“View Deal”, web_app=WebAppInfo(url=f”{url}?deal={uid}”))
]])
await msg.answer(“You have been invited to a secure escrow deal. Tap to view it.”, reply_markup=kb)
return
kb = InlineKeyboardMarkup(inline_keyboard=[
[InlineKeyboardButton(text=“Open EscrowBot”, web_app=WebAppInfo(url=url))],
[InlineKeyboardButton(text=“Create Deal”,    web_app=WebAppInfo(url=f”{url}?page=create”))],
])
await msg.answer(
“Welcome to EscrowBot! Secure middleman for online deals.\n\nTap below to start:”,
reply_markup=kb
)

@bot_router.message(Command(“help”))
async def cmd_help(msg: Message):
await msg.answer(”/start - Open the app\n/help - This message”)

def setup_bot() -> tuple[Bot, Dispatcher]:
b = Bot(token=BOT_TOKEN) if BOT_TOKEN else None
d = Dispatcher()
d.include_router(bot_router)
return b, d

# —————————————————————————

# HTML Frontend (React via CDN + in-browser Babel)

# —————————————————————————

HTML_APP = “””<!DOCTYPE html>

<html lang="en">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no"/>
<title>EscrowBot</title>
<script src="https://telegram.org/js/telegram-web-app.js"></script>
<script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
<script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
<script src="https://cdn.tailwindcss.com"></script>
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link href="https://fonts.googleapis.com/css2?family=Syne:wght@400;600;700;800&family=JetBrains+Mono:wght@400;600&display=swap" rel="stylesheet"/>
<script>
tailwind.config = {
  theme: {
    extend: {
      colors: {
        ng:  "#00ff88",
        np:  "#b84fff",
        nb:  "#00d4ff",
        d9:  "#030712",
        d8:  "#0a0f1e",
        d7:  "#0f1629",
        d6:  "#151f3a"
      },
      fontFamily: {
        syne: ["Syne","sans-serif"],
        mono: ["JetBrains Mono","monospace"]
      }
    }
  }
}
</script>
<style>
*{box-sizing:border-box;margin:0;padding:0;-webkit-tap-highlight-color:transparent}
body{background:#030712;color:#fff;font-family:'Syne',sans-serif;overflow-x:hidden;-webkit-font-smoothing:antialiased}
::-webkit-scrollbar{width:3px}
::-webkit-scrollbar-thumb{background:rgba(255,255,255,.1);border-radius:2px}
.glass{background:rgba(255,255,255,.05);backdrop-filter:blur(16px);-webkit-backdrop-filter:blur(16px);border:1px solid rgba(255,255,255,.1)}
.glass-green{background:rgba(0,255,136,.07);border:1px solid rgba(0,255,136,.2)}
.glass-purple{background:rgba(184,79,255,.07);border:1px solid rgba(184,79,255,.2)}
.glass-blue{background:rgba(0,212,255,.07);border:1px solid rgba(0,212,255,.2)}
.glass-red{background:rgba(239,68,68,.07);border:1px solid rgba(239,68,68,.2)}
.glow-green{box-shadow:0 0 24px rgba(0,255,136,.25)}
.glow-purple{box-shadow:0 0 24px rgba(184,79,255,.25)}
.glow-blue{box-shadow:0 0 24px rgba(0,212,255,.25)}
.neon-text{background:linear-gradient(135deg,#00ff88,#00d4ff);-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text}
@keyframes fadeUp{from{opacity:0;transform:translateY(18px)}to{opacity:1;transform:translateY(0)}}
@keyframes pulse-dot{0%,100%{opacity:1}50%{opacity:.4}}
@keyframes spin{to{transform:rotate(360deg)}}
@keyframes shimmer{0%{transform:translateX(-100%)}100%{transform:translateX(100%)}}
@keyframes slideUp{from{transform:translateY(100%);opacity:0}to{transform:translateY(0);opacity:1}}
@keyframes fadeIn{from{opacity:0}to{opacity:1}}
.fade-up{animation:fadeUp .4s ease both}
.slide-up{animation:slideUp .35s cubic-bezier(.32,.72,0,1) both}
.fade-in{animation:fadeIn .25s ease both}
.spin{animation:spin .8s linear infinite}
.pulse-dot{animation:pulse-dot 1.5s ease infinite}
.delay-1{animation-delay:.08s}
.delay-2{animation-delay:.16s}
.delay-3{animation-delay:.24s}
.delay-4{animation-delay:.32s}
.btn{display:flex;align-items:center;justify-content:center;gap:6px;border-radius:14px;padding:12px 18px;font-size:13px;font-weight:700;font-family:'Syne',sans-serif;cursor:pointer;border:none;transition:all .15s;letter-spacing:.02em}
.btn:active{transform:scale(.95)}
.btn-solid{background:#00ff88;color:#030712}
.btn-solid:hover{background:#00e87a}
.btn-green{background:rgba(0,255,136,.12);border:1px solid rgba(0,255,136,.35);color:#00ff88}
.btn-purple{background:rgba(184,79,255,.12);border:1px solid rgba(184,79,255,.35);color:#b84fff}
.btn-blue{background:rgba(0,212,255,.12);border:1px solid rgba(0,212,255,.35);color:#00d4ff}
.btn-red{background:rgba(239,68,68,.12);border:1px solid rgba(239,68,68,.35);color:#f87171}
.btn-ghost{background:rgba(255,255,255,.06);border:1px solid rgba(255,255,255,.12);color:rgba(255,255,255,.6)}
.btn:disabled{opacity:.35;cursor:not-allowed;transform:none}
.input{width:100%;background:rgba(255,255,255,.05);border:1px solid rgba(255,255,255,.1);border-radius:13px;padding:12px 14px;color:#fff;font-family:'Syne',sans-serif;font-size:14px;outline:none;transition:border-color .2s}
.input:focus{border-color:rgba(0,255,136,.45)}
.input::placeholder{color:rgba(255,255,255,.25)}
.overlay{position:fixed;inset:0;background:rgba(0,0,0,.65);backdrop-filter:blur(6px);z-index:40;animation:fadeIn .2s ease}
.sheet{position:fixed;bottom:0;left:0;right:0;z-index:50;background:#0a0f1e;border:1px solid rgba(255,255,255,.08);border-bottom:none;border-radius:24px 24px 0 0;padding:20px;padding-bottom:32px;animation:slideUp .35s cubic-bezier(.32,.72,0,1)}
.handle{width:40px;height:4px;background:rgba(255,255,255,.15);border-radius:2px;margin:0 auto 20px}
.badge{display:inline-flex;align-items:center;gap:5px;padding:3px 10px;border-radius:99px;font-size:11px;font-weight:700;font-family:'Syne',sans-serif}
.toast{position:fixed;top:16px;left:50%;transform:translateX(-50%);z-index:999;background:#0f1629;border:1px solid rgba(255,255,255,.12);border-radius:14px;padding:10px 18px;font-size:13px;font-weight:600;white-space:nowrap;animation:fadeUp .3s ease}
</style>
</head>
<body>
<div id="root"></div>
<script type="text/babel">
const { useState, useEffect, useRef, useCallback } = React;

// ── Constants ──────────────────────────────────────────────────────────────
const API = “”;  // same origin
const BOT_USER = window.**BOT_USERNAME** || “EscrowBot”;

const T = {
en: {
dir:“ltr”, welcome:“EscrowBot”, tagline:“Secure trades. Zero trust required.”,
home:“Home”, trades:“Trades”, wallet:“Wallet”,
createDeal:“Create Deal”, myDeals:“My Deals”,
activeTrades:“Active”, totalVol:“Volume”, reputation:“Rating”,
pending:“Pending”, funded:“Funded”, delivered:“Delivered”,
completed:“Completed”, disputed:“Disputed”, cancelled:“Cancelled”,
fundEscrow:“Fund Escrow”, markDelivered:“Mark Delivered”,
confirmReceipt:“Confirm Receipt”, raiseDispute:“Raise Dispute”,
shareLink:“Share Link”, copyLink:“Copy Link”, copied:“Copied!”,
dealTitle:“Deal Title”, dealDesc:“Description”, dealAmount:“Amount (USD)”,
platformFee:“Platform Fee”, total:“Total Payable”,
create:“Create Deal”, submitting:“Submitting…”,
disputeReason:“Reason for Dispute”, submitDispute:“Submit Dispute”,
proofLabel:“Delivery Proof / Link”, submitProof:“Mark as Delivered”,
noDeals:“No deals yet”, startMsg:“Create your first deal to start trading”,
feeHint:“Fee: 1.5% for deals $50+ | $1 flat for deals under $50”,
ar:“العربية”, en:“English”,
loading:“Loading…”, error:“Something went wrong”,
sellerTag:“Seller”, buyerTag:“Buyer”, feeTag:“Fee”,
},
ar: {
dir:“rtl”, welcome:“EscrowBot”, tagline:“صفقات آمنة. لا حاجة للثقة.”,
home:“الرئيسية”, trades:“الصفقات”, wallet:“المحفظة”,
createDeal:“إنشاء صفقة”, myDeals:“صفقاتي”,
activeTrades:“النشطة”, totalVol:“الحجم”, reputation:“التقييم”,
pending:“معلقة”, funded:“ممولة”, delivered:“مُسلَّمة”,
completed:“مكتملة”, disputed:“متنازع”, cancelled:“ملغاة”,
fundEscrow:“تمويل الضمان”, markDelivered:“تحديد مُسلَّم”,
confirmReceipt:“تأكيد الاستلام”, raiseDispute:“رفع نزاع”,
shareLink:“مشاركة الرابط”, copyLink:“نسخ الرابط”, copied:“تم النسخ!”,
dealTitle:“عنوان الصفقة”, dealDesc:“الوصف”, dealAmount:“المبلغ (دولار)”,
platformFee:“رسوم المنصة”, total:“الإجمالي”,
create:“إنشاء الصفقة”, submitting:“جاري الإرسال…”,
disputeReason:“سبب النزاع”, submitDispute:“تقديم النزاع”,
proofLabel:“دليل التسليم / الرابط”, submitProof:“تحديد كمُسلَّم”,
noDeals:“لا توجد صفقات”, startMsg:“أنشئ صفقتك الأولى للبدء”,
feeHint:“الرسوم: 1.5% للصفقات فوق $50 | $1 ثابت لما دون $50”,
ar:“العربية”, en:“English”,
loading:“جاري التحميل…”, error:“حدث خطأ ما”,
sellerTag:“البائع”, buyerTag:“المشتري”, feeTag:“الرسوم”,
}
};

// ── Toast ──────────────────────────────────────────────────────────────────
let _setToast = null;
function Toast() {
const [msg, setMsg] = useState(null);
_setToast = setMsg;
useEffect(() => {
if (!msg) return;
const id = setTimeout(() => setMsg(null), 2800);
return () => clearTimeout(id);
}, [msg]);
if (!msg) return null;
return <div className="toast">{msg.icon} {msg.text}</div>;
}
function toast(text, icon=“✓”) { _setToast && _setToast({text,icon}); }
function toastErr(text)         { _setToast && _setToast({text, icon:“✕”}); }

// ── Status badge ───────────────────────────────────────────────────────────
const statusStyle = {
pending:   {bg:“rgba(250,204,21,.12)”,  border:“rgba(250,204,21,.3)”,  color:”#fde68a”, dot:”#fbbf24”},
funded:    {bg:“rgba(0,212,255,.12)”,   border:“rgba(0,212,255,.3)”,   color:”#00d4ff”, dot:”#00d4ff”},
delivered: {bg:“rgba(184,79,255,.12)”,  border:“rgba(184,79,255,.3)”,  color:”#b84fff”, dot:”#b84fff”},
completed: {bg:“rgba(0,255,136,.12)”,   border:“rgba(0,255,136,.3)”,   color:”#00ff88”, dot:”#00ff88”},
disputed:  {bg:“rgba(239,68,68,.12)”,   border:“rgba(239,68,68,.3)”,   color:”#f87171”, dot:”#f87171”},
cancelled: {bg:“rgba(156,163,175,.12)”, border:“rgba(156,163,175,.3)”, color:”#9ca3af”, dot:”#9ca3af”},
};
function StatusBadge({status, label}) {
const s = statusStyle[status] || statusStyle.pending;
return (
<span className=“badge” style={{background:s.bg, border:`1px solid ${s.border}`, color:s.color}}>
<span className="pulse-dot" style={{width:6,height:6,borderRadius:9999,background:s.dot,flexShrink:0}}/>
{label}
</span>
);
}

// ── API calls ──────────────────────────────────────────────────────────────
function initData() {
return window.Telegram?.WebApp?.initData || “”;
}
async function apiFetch(path, opts={}) {
const res = await fetch(API + “/api” + path, {
headers: {“Content-Type”:“application/json”, “x-init-data”: initData()},
…opts,
});
if (!res.ok) {
const err = await res.json().catch(() => ({}));
throw new Error(err.detail || “API error”);
}
return res.json();
}

// ── Spinner ────────────────────────────────────────────────────────────────
function Spinner({color=”#00ff88”}) {
return <span style={{width:16,height:16,border:`2px solid ${color}30`,borderTopColor:color,borderRadius:9999,display:“inline-block”}} className=“spin”/>;
}

// ── Modal wrapper ──────────────────────────────────────────────────────────
function Modal({onClose, children}) {
return (
<>
<div className="overlay" onClick={onClose}/>
<div className="sheet">
<div className="handle"/>
{children}
</div>
</>
);
}

// ── Create Deal Modal ──────────────────────────────────────────────────────
function CreateDealModal({t, onClose, onCreated}) {
const [form, setForm] = useState({title:””, description:””, amount:””, currency:“USD”});
const [feeInfo, setFeeInfo] = useState(null);
const [loading, setLoading] = useState(false);

useEffect(() => {
const a = parseFloat(form.amount);
if (!a || a <= 0) { setFeeInfo(null); return; }
const timer = setTimeout(async () => {
try {
const d = await apiFetch(`/fee-info?amount=${a}`);
setFeeInfo(d);
} catch {}
}, 400);
return () => clearTimeout(timer);
}, [form.amount]);

const submit = async () => {
if (!form.title || !form.amount) return;
setLoading(true);
try {
const deal = await apiFetch(”/deals”, {
method:“POST”,
body: JSON.stringify({
title: form.title,
description: form.description || null,
amount: parseFloat(form.amount),
currency: form.currency,
})
});
toast(t.dealCreated || “Deal created!”);
onCreated(deal);
onClose();
} catch(e) { toastErr(e.message); }
finally { setLoading(false); }
};

return (
<Modal onClose={onClose}>
<div style={{display:“flex”,alignItems:“center”,justifyContent:“space-between”,marginBottom:16}}>
<h2 style={{fontSize:17,fontWeight:800}}>{t.createDeal}</h2>
<button onClick={onClose} style={{width:30,height:30,borderRadius:99,background:“rgba(255,255,255,.08)”,border:“none”,color:“rgba(255,255,255,.5)”,fontSize:16,cursor:“pointer”}}>✕</button>
</div>
<div style={{display:“flex”,flexDirection:“column”,gap:10}}>
<div>
<div style={{fontSize:11,color:“rgba(255,255,255,.45)”,marginBottom:5}}>{t.dealTitle}</div>
<input className=“input” placeholder=“e.g. Logo Design, Script…” value={form.title} onChange={e=>setForm(f=>({…f,title:e.target.value}))} maxLength={256}/>
</div>
<div>
<div style={{fontSize:11,color:“rgba(255,255,255,.45)”,marginBottom:5}}>{t.dealDesc}</div>
<textarea className=“input” rows={2} style={{resize:“none”}} placeholder=“Describe the deal…” value={form.description} onChange={e=>setForm(f=>({…f,description:e.target.value}))}/>
</div>
<div style={{display:“grid”,gridTemplateColumns:“1fr auto”,gap:8}}>
<div>
<div style={{fontSize:11,color:“rgba(255,255,255,.45)”,marginBottom:5}}>{t.dealAmount}</div>
<input className=“input” type=“number” min=“0.01” step=“0.01” placeholder=“0.00” value={form.amount} onChange={e=>setForm(f=>({…f,amount:e.target.value}))}/>
</div>
<div>
<div style={{fontSize:11,color:“rgba(255,255,255,.45)”,marginBottom:5}}>Currency</div>
<select className=“input” style={{width:80}} value={form.currency} onChange={e=>setForm(f=>({…f,currency:e.target.value}))}>
<option>USD</option><option>EUR</option><option>USDT</option>
</select>
</div>
</div>
{feeInfo && (
<div className=“glass-green” style={{borderRadius:12,padding:“10px 14px”,fontSize:12}}>
<div style={{color:”#00ff88”,fontWeight:700,marginBottom:6}}>⚡ Fee Breakdown</div>
<div style={{display:“flex”,justifyContent:“space-between”,color:“rgba(255,255,255,.5)”}}>
<span>{t.dealAmount}</span><span className="font-mono">${feeInfo.amount.toFixed(2)}</span>
</div>
<div style={{display:“flex”,justifyContent:“space-between”,color:”#fbbf24”}}>
<span>{t.platformFee} ({feeInfo.fee_type === “percentage” ? “1.5%” : “$1 flat”})</span>
<span className="font-mono">${feeInfo.fee.toFixed(2)}</span>
</div>
<div style={{height:1,background:“rgba(255,255,255,.08)”,margin:“6px 0”}}/>
<div style={{display:“flex”,justifyContent:“space-between”,fontWeight:800,color:”#00ff88”}}>
<span>{t.total}</span><span className="font-mono">${feeInfo.total.toFixed(2)}</span>
</div>
</div>
)}
<button className=“btn btn-solid” onClick={submit} disabled={loading||!form.title||!form.amount} style={{marginTop:4,width:“100%”}}>
{loading ? <Spinner color="#030712"/> : null}
{loading ? t.submitting : t.create}
</button>
<p style={{fontSize:11,color:“rgba(255,255,255,.25)”,textAlign:“center”}}>{t.feeHint}</p>
</div>
</Modal>
);
}

// ── Dispute Modal ──────────────────────────────────────────────────────────
function DisputeModal({t, deal, onClose, onUpdated}) {
const [reason, setReason] = useState(””);
const [loading, setLoading] = useState(false);
const submit = async () => {
setLoading(true);
try {
const d = await apiFetch(`/deals/${deal.deal_uid}/dispute`, {method:“POST”, body:JSON.stringify({reason})});
toast(t.disputeRaised || “Dispute raised”);
onUpdated(d); onClose();
} catch(e) { toastErr(e.message); }
finally { setLoading(false); }
};
return (
<Modal onClose={onClose}>
<div style={{display:“flex”,alignItems:“center”,gap:8,marginBottom:14}}>
<span style={{fontSize:18}}>⚠️</span>
<h2 style={{fontSize:17,fontWeight:800}}>{t.raiseDispute}</h2>
<button onClick={onClose} style={{marginLeft:“auto”,width:30,height:30,borderRadius:99,background:“rgba(255,255,255,.08)”,border:“none”,color:“rgba(255,255,255,.5)”,fontSize:16,cursor:“pointer”}}>✕</button>
</div>
<div className=“glass-red” style={{borderRadius:12,padding:“8px 12px”,marginBottom:12,fontSize:12,color:”#f87171”}}>
Disputes are reviewed within 24-48 hours. Please be specific.
</div>
<div style={{marginBottom:10}}>
<div style={{fontSize:11,color:“rgba(255,255,255,.45)”,marginBottom:5}}>{t.disputeReason}</div>
<textarea className=“input” rows={3} style={{resize:“none”}} placeholder=“Explain the issue clearly…” value={reason} onChange={e=>setReason(e.target.value)}/>
</div>
<button className=“btn btn-red” onClick={submit} disabled={loading||reason.length<10} style={{width:“100%”}}>
{loading ? <Spinner color="#f87171"/> : null} {t.submitDispute}
</button>
</Modal>
);
}

// ── Delivery Proof Modal ───────────────────────────────────────────────────
function ProofModal({t, deal, onClose, onUpdated}) {
const [proof, setProof] = useState(””);
const [loading, setLoading] = useState(false);
const submit = async () => {
setLoading(true);
try {
const d = await apiFetch(`/deals/${deal.deal_uid}/deliver`, {method:“POST”, body:JSON.stringify({proof})});
toast(t.deliveryMarked || “Marked as delivered!”);
onUpdated(d); onClose();
} catch(e) { toastErr(e.message); }
finally { setLoading(false); }
};
return (
<Modal onClose={onClose}>
<div style={{display:“flex”,alignItems:“center”,justifyContent:“space-between”,marginBottom:14}}>
<h2 style={{fontSize:17,fontWeight:800}}>{t.markDelivered}</h2>
<button onClick={onClose} style={{width:30,height:30,borderRadius:99,background:“rgba(255,255,255,.08)”,border:“none”,color:“rgba(255,255,255,.5)”,fontSize:16,cursor:“pointer”}}>✕</button>
</div>
<div style={{marginBottom:10}}>
<div style={{fontSize:11,color:“rgba(255,255,255,.45)”,marginBottom:5}}>{t.proofLabel}</div>
<textarea className=“input” rows={3} style={{resize:“none”}} placeholder=“Paste delivery link, credentials, or proof…” value={proof} onChange={e=>setProof(e.target.value)}/>
</div>
<button className=“btn btn-purple” onClick={submit} disabled={loading||proof.length<5} style={{width:“100%”}}>
{loading ? <Spinner color="#b84fff"/> : null} {t.submitProof}
</button>
</Modal>
);
}

// ── Share Modal ────────────────────────────────────────────────────────────
function ShareModal({t, deal, onClose}) {
const [copied, setCopied] = useState(false);
const link = `https://t.me/${BOT_USER}?start=deal_${deal.deal_uid}`;
const copy = async () => {
try { await navigator.clipboard.writeText(link); }
catch {
const el = document.createElement(“textarea”);
el.value = link; document.body.appendChild(el);
el.select(); document.execCommand(“copy”);
document.body.removeChild(el);
}
setCopied(true); toast(t.copied);
setTimeout(() => setCopied(false), 2000);
};
const share = () => {
const text = encodeURIComponent(`Join my secure escrow deal: ${deal.title}`);
window.Telegram?.WebApp?.openTelegramLink(`https://t.me/share/url?url=${encodeURIComponent(link)}&text=${text}`);
};
return (
<Modal onClose={onClose}>
<div style={{display:“flex”,alignItems:“center”,justifyContent:“space-between”,marginBottom:14}}>
<h2 style={{fontSize:17,fontWeight:800}}>{t.shareLink}</h2>
<button onClick={onClose} style={{width:30,height:30,borderRadius:99,background:“rgba(255,255,255,.08)”,border:“none”,color:“rgba(255,255,255,.5)”,fontSize:16,cursor:“pointer”}}>✕</button>
</div>
<div className=“glass” style={{borderRadius:12,padding:“10px 14px”,marginBottom:12}}>
<p style={{fontSize:11,color:“rgba(255,255,255,.4)”,marginBottom:4}}>{deal.title}</p>
<p style={{fontSize:11,color:”#00ff88”,fontFamily:”‘JetBrains Mono’,monospace”,wordBreak:“break-all”}}>{link}</p>
</div>
<div style={{display:“flex”,gap:8}}>
<button className="btn btn-ghost" onClick={copy} style={{flex:1}}>
{copied ? “✓ “+t.copied : “📋 “+t.copyLink}
</button>
<button className="btn btn-green" onClick={share} style={{flex:1}}>
✈️ Telegram
</button>
</div>
</Modal>
);
}

// ── Deal Card ──────────────────────────────────────────────────────────────
function DealCard({deal, t, tgUserId, onUpdate}) {
const [open, setOpen] = useState(false);
const [modal, setModal] = useState(null);
const [acting, setActing] = useState(false);

const isSeller = tgUserId && deal.seller_id === tgUserId;
const isBuyer  = tgUserId && deal.buyer_id  === tgUserId;

const glowMap = {funded:“glass-blue”, delivered:“glass-purple”, completed:“glass-green”};
const cardClass = glowMap[deal.status] || “glass”;

const fmt = new Intl.NumberFormat(“en-US”,{style:“currency”,currency:deal.currency||“USD”}).format(deal.amount);

const doAction = async (action) => {
setActing(true);
try {
let d;
if (action === “fund”)    d = await apiFetch(`/deals/${deal.deal_uid}/fund`,    {method:“POST”, body:JSON.stringify({})});
if (action === “confirm”) d = await apiFetch(`/deals/${deal.deal_uid}/confirm`, {method:“POST”});
if (d) { onUpdate(d); toast({fund:“Escrow funded! 🔒”, confirm:“Receipt confirmed! 💸”}[action]); }
} catch(e) { toastErr(e.message); }
finally { setActing(false); }
};

return (
<>
<div className={cardClass} style={{borderRadius:18,marginBottom:10,overflow:“hidden”,transition:“all .2s”}}>
{/* Header row */}
<div onClick={()=>setOpen(v=>!v)} style={{padding:“14px 16px”,display:“flex”,alignItems:“flex-start”,justifyContent:“space-between”,cursor:“pointer”}}>
<div style={{flex:1,minWidth:0}}>
<StatusBadge status={deal.status} label={t[deal.status]||deal.status}/>
<p style={{fontSize:14,fontWeight:700,marginTop:6,whiteSpace:“nowrap”,overflow:“hidden”,textOverflow:“ellipsis”}}>{deal.title}</p>
<p style={{fontSize:11,color:“rgba(255,255,255,.35)”,fontFamily:”‘JetBrains Mono’,monospace”,marginTop:2}}>#{deal.deal_uid?.slice(0,8)}</p>
</div>
<div style={{display:“flex”,flexDirection:“column”,alignItems:“flex-end”,gap:4,marginLeft:12}}>
<span style={{color:”#00ff88”,fontWeight:800,fontFamily:”‘JetBrains Mono’,monospace”,fontSize:15}}>{fmt}</span>
<span style={{fontSize:14,color:“rgba(255,255,255,.3)”,transition:“transform .2s”,transform:open?“rotate(180deg)”:“none”}}>⌄</span>
</div>
</div>

```
    {/* Expanded */}
    {open && (
      <div style={{padding:"0 16px 16px",borderTop:"1px solid rgba(255,255,255,.06)"}}>
        {deal.description && <p style={{fontSize:12,color:"rgba(255,255,255,.4)",marginTop:10,marginBottom:10}}>{deal.description}</p>}
        {/* Stats */}
        <div style={{display:"grid",gridTemplateColumns:"1fr 1fr 1fr",gap:6,marginBottom:12}}>
          {[
            {label:t.feeTag, val:`$${deal.fee}`, color:"#fbbf24"},
            {label:t.sellerTag, val:`#${String(deal.seller_id).slice(-4)}`, color:"rgba(255,255,255,.6)"},
            {label:t.buyerTag, val: deal.buyer_id ? `#${String(deal.buyer_id).slice(-4)}` : "—", color:"rgba(255,255,255,.6)"},
          ].map(s=>(
            <div key={s.label} className="glass" style={{borderRadius:10,padding:"8px",textAlign:"center"}}>
              <p style={{fontSize:10,color:"rgba(255,255,255,.35)",marginBottom:3}}>{s.label}</p>
              <p style={{fontSize:12,fontWeight:700,color:s.color,fontFamily:"'JetBrains Mono',monospace"}}>{s.val}</p>
            </div>
          ))}
        </div>

        {/* Actions */}
        <div style={{display:"flex",flexDirection:"column",gap:7}}>
          {deal.status==="pending" && !isSeller && (
            <button className="btn btn-solid" onClick={()=>doAction("fund")} disabled={acting} style={{width:"100%"}}>
              {acting?<Spinner color="#030712"/>:null} {t.fundEscrow}
            </button>
          )}
          {deal.status==="funded" && isSeller && (
            <button className="btn btn-purple" onClick={()=>setModal("proof")} style={{width:"100%"}}>
              {t.markDelivered}
            </button>
          )}
          {deal.status==="delivered" && isBuyer && (
            <>
              <button className="btn btn-solid" onClick={()=>doAction("confirm")} disabled={acting} style={{width:"100%"}}>
                {acting?<Spinner color="#030712"/>:null} {t.confirmReceipt}
              </button>
              <button className="btn btn-red" onClick={()=>setModal("dispute")} style={{width:"100%"}}>
                {t.raiseDispute}
              </button>
            </>
          )}
          {deal.status==="funded" && isBuyer && (
            <button className="btn btn-red" onClick={()=>setModal("dispute")} style={{width:"100%"}}>
              {t.raiseDispute}
            </button>
          )}
          {deal.status==="pending" && isSeller && (
            <button className="btn btn-ghost" onClick={()=>setModal("share")} style={{width:"100%"}}>
              🔗 {t.shareLink}
            </button>
          )}
        </div>
      </div>
    )}
  </div>

  {modal==="dispute" && <DisputeModal t={t} deal={deal} onClose={()=>setModal(null)} onUpdated={d=>{onUpdate(d);setModal(null);}}/>}
  {modal==="proof"   && <ProofModal   t={t} deal={deal} onClose={()=>setModal(null)} onUpdated={d=>{onUpdate(d);setModal(null);}}/>}
  {modal==="share"   && <ShareModal   t={t} deal={deal} onClose={()=>setModal(null)}/>}
</>
```

);
}

// ── Home Tab ───────────────────────────────────────────────────────────────
function HomeTab({t, user, deals, onOpenCreate, onSwitchTab}) {
const active    = deals.filter(d=>![“completed”,“cancelled”].includes(d.status));
const completed = deals.filter(d=>d.status===“completed”);
const volume    = completed.reduce((s,d)=>s+d.amount, 0);

return (
<div style={{padding:“0 16px 100px”}}>
{/* Glow blobs */}
<div style={{position:“fixed”,top:0,left:“30%”,width:300,height:300,background:“rgba(0,255,136,.04)”,borderRadius:9999,filter:“blur(60px)”,pointerEvents:“none”}}/>
<div style={{position:“fixed”,top:80,right:”-10%”,width:200,height:200,background:“rgba(184,79,255,.04)”,borderRadius:9999,filter:“blur(50px)”,pointerEvents:“none”}}/>

```
  {/* Header */}
  <div className="fade-up" style={{marginBottom:20}}>
    <p style={{fontSize:13,color:"rgba(255,255,255,.35)",marginBottom:2}}>
      {user ? `Welcome, ${user.first_name}` : "Welcome"}
    </p>
    <h1 style={{fontSize:28,fontWeight:800}} className="neon-text">EscrowBot</h1>
    <p style={{fontSize:13,color:"rgba(255,255,255,.35)",marginTop:2}}>{t.tagline}</p>
  </div>

  {/* Stats */}
  <div className="fade-up delay-1" style={{display:"grid",gridTemplateColumns:"1fr 1fr 1fr",gap:10,marginBottom:16}}>
    {[
      {label:t.activeTrades, val:active.length,  color:"#00d4ff", icon:"⚡"},
      {label:t.totalVol,     val:`$${volume.toFixed(0)}`, color:"#00ff88", icon:"📈"},
      {label:t.reputation,   val:(user?.reputation_score||5).toFixed(1), color:"#fbbf24", icon:"⭐"},
    ].map(s=>(
      <div key={s.label} className="glass" style={{borderRadius:16,padding:"12px 8px",textAlign:"center"}}>
        <div style={{fontSize:18,marginBottom:4}}>{s.icon}</div>
        <div style={{fontSize:15,fontWeight:800,color:s.color,fontFamily:"'JetBrains Mono',monospace"}}>{s.val}</div>
        <div style={{fontSize:10,color:"rgba(255,255,255,.35)",marginTop:2}}>{s.label}</div>
      </div>
    ))}
  </div>

  {/* Actions */}
  <div className="fade-up delay-2" style={{display:"grid",gridTemplateColumns:"1fr 1fr",gap:10,marginBottom:20}}>
    <button className="btn btn-solid" onClick={onOpenCreate} style={{padding:"15px",fontSize:14}}>
      + {t.createDeal}
    </button>
    <button className="btn btn-blue" onClick={()=>onSwitchTab("trades")} style={{padding:"15px",fontSize:14}}>
      {t.myDeals}
    </button>
  </div>

  {/* Recent deals preview */}
  {active.length > 0 ? (
    <div className="fade-up delay-3">
      <p style={{fontSize:11,fontWeight:700,color:"rgba(255,255,255,.3)",letterSpacing:"0.12em",textTransform:"uppercase",marginBottom:10}}>
        Active Deals
      </p>
      {active.slice(0,3).map(d=>(
        <div key={d.deal_uid} className="glass" onClick={()=>onSwitchTab("trades")}
          style={{borderRadius:14,padding:"12px 14px",marginBottom:8,display:"flex",justifyContent:"space-between",cursor:"pointer"}}>
          <div>
            <p style={{fontSize:13,fontWeight:700,maxWidth:200,overflow:"hidden",textOverflow:"ellipsis",whiteSpace:"nowrap"}}>{d.title}</p>
            <p style={{fontSize:11,color:"rgba(255,255,255,.3)",fontFamily:"'JetBrains Mono',monospace"}}>#{d.deal_uid?.slice(0,8)}</p>
          </div>
          <span style={{color:"#00ff88",fontWeight:800,fontFamily:"'JetBrains Mono',monospace",fontSize:13}}>${d.amount}</span>
        </div>
      ))}
    </div>
  ) : (
    <div className="fade-up delay-3 glass" style={{borderRadius:20,padding:"36px 24px",textAlign:"center"}}>
      <div style={{fontSize:40,marginBottom:12}}>🔐</div>
      <p style={{fontWeight:700,color:"rgba(255,255,255,.5)",marginBottom:6}}>{t.noDeals}</p>
      <p style={{fontSize:12,color:"rgba(255,255,255,.25)"}}>{t.startMsg}</p>
    </div>
  )}
</div>
```

);
}

// ── Trades Tab ─────────────────────────────────────────────────────────────
function TradesTab({t, deals, tgUserId, onUpdate, onRefresh, loading}) {
const [filter, setFilter] = useState(“all”);
const filters = [
{k:“all”,label:“All”},
{k:“active”,label:“Active”},
{k:“completed”,label:“Done”},
{k:“disputed”,label:“Disputed”},
];
const shown = deals.filter(d=>{
if(filter===“all”) return true;
if(filter===“active”) return [“pending”,“funded”,“delivered”].includes(d.status);
if(filter===“completed”) return d.status===“completed”;
if(filter===“disputed”) return d.status===“disputed”;
return true;
});

return (
<div style={{padding:“0 16px 100px”}}>
<div style={{display:“flex”,alignItems:“center”,justifyContent:“space-between”,marginBottom:16}}>
<h1 style={{fontSize:20,fontWeight:800}}>{t.trades}</h1>
<button onClick={onRefresh} style={{width:34,height:34,borderRadius:99,background:“rgba(255,255,255,.07)”,border:“1px solid rgba(255,255,255,.1)”,color:“rgba(255,255,255,.5)”,cursor:“pointer”,fontSize:14}}>
{loading ? <Spinner/> : “↻”}
</button>
</div>
{/* Filter pills */}
<div style={{display:“flex”,gap:7,marginBottom:14,overflowX:“auto”,paddingBottom:4}}>
{filters.map(f=>(
<button key={f.k} onClick={()=>setFilter(f.k)}
style={{
padding:“6px 14px”,borderRadius:99,fontSize:12,fontWeight:700,cursor:“pointer”,
whiteSpace:“nowrap”,transition:“all .15s”,
background: filter===f.k ? “rgba(0,255,136,.15)” : “rgba(255,255,255,.05)”,
border: filter===f.k ? “1px solid rgba(0,255,136,.35)” : “1px solid rgba(255,255,255,.08)”,
color: filter===f.k ? “#00ff88” : “rgba(255,255,255,.4)”,
}}>{f.label}</button>
))}
</div>
{loading ? (
<div style={{textAlign:“center”,padding:48}}><Spinner/></div>
) : shown.length === 0 ? (
<div style={{textAlign:“center”,padding:48,color:“rgba(255,255,255,.3)”,fontSize:14}}>{t.noDeals}</div>
) : (
shown.map(d=><DealCard key={d.deal_uid} deal={d} t={t} tgUserId={tgUserId} onUpdate={onUpdate}/>)
)}
</div>
);
}

// ── Wallet Tab ─────────────────────────────────────────────────────────────
function WalletTab({t, user, deals, tgUserId}) {
const locked  = deals.filter(d=>[“funded”,“delivered”].includes(d.status)&&d.buyer_id===tgUserId).reduce((s,d)=>s+d.amount+d.fee,0);
const earned  = deals.filter(d=>d.status===“completed”&&d.seller_id===tgUserId).reduce((s,d)=>s+d.amount,0);
return (
<div style={{padding:“0 16px 100px”}}>
<div style={{position:“fixed”,top:0,left:0,right:0,height:240,background:“linear-gradient(180deg,rgba(184,79,255,.08) 0%,transparent 100%)”,pointerEvents:“none”}}/>
<h1 className="fade-up" style={{fontSize:20,fontWeight:800,marginBottom:16}}>{t.wallet}</h1>
{/* Balance card */}
<div className=“fade-up delay-1 glass-purple glow-purple” style={{borderRadius:22,padding:“28px 24px”,textAlign:“center”,marginBottom:14}}>
<div style={{width:56,height:56,borderRadius:18,background:“rgba(184,79,255,.2)”,display:“flex”,alignItems:“center”,justifyContent:“center”,margin:“0 auto 12px”,fontSize:24}}>💜</div>
<p style={{fontSize:12,color:“rgba(255,255,255,.4)”,marginBottom:4}}>Available Balance</p>
<p style={{fontSize:38,fontWeight:800,fontFamily:”‘JetBrains Mono’,monospace”}}>${(user?.wallet_balance||0).toFixed(2)}</p>
<p style={{fontSize:12,color:“rgba(255,255,255,.25)”,marginTop:4}}>USD</p>
</div>
{/* Stats */}
<div className=“fade-up delay-2” style={{display:“grid”,gridTemplateColumns:“1fr 1fr”,gap:10,marginBottom:14}}>
<div className=“glass” style={{borderRadius:16,padding:“16px”}}>
<div style={{fontSize:12,color:“rgba(255,255,255,.35)”,display:“flex”,alignItems:“center”,gap:5,marginBottom:6}}>🔒 In Escrow</div>
<div style={{fontSize:20,fontWeight:800,color:”#fbbf24”,fontFamily:”‘JetBrains Mono’,monospace”}}>${locked.toFixed(2)}</div>
</div>
<div className=“glass” style={{borderRadius:16,padding:“16px”}}>
<div style={{fontSize:12,color:“rgba(255,255,255,.35)”,display:“flex”,alignItems:“center”,gap:5,marginBottom:6}}>💸 Total Earned</div>
<div style={{fontSize:20,fontWeight:800,color:”#00ff88”,fontFamily:”‘JetBrains Mono’,monospace”}}>${earned.toFixed(2)}</div>
</div>
</div>
<div className=“fade-up delay-3 glass” style={{borderRadius:14,padding:“14px”,borderStyle:“dashed”,textAlign:“center”}}>
<p style={{fontSize:12,color:“rgba(255,255,255,.25)”}}>Crypto deposit & withdrawal coming soon. Balance is simulated.</p>
</div>
</div>
);
}

// ── Bottom Nav ─────────────────────────────────────────────────────────────
function BottomNav({t, active, onChange}) {
const tabs = [
{id:“home”,   icon:“🏠”, label:t.home},
{id:“trades”, icon:“🔄”, label:t.trades},
{id:“wallet”, icon:“💼”, label:t.wallet},
];
return (
<div style={{
position:“fixed”,bottom:0,left:0,right:0,zIndex:30,
padding:“8px 16px 16px”,
background:“linear-gradient(to top, rgba(3,7,18,.95) 60%, transparent)”
}}>
<div className=“glass” style={{
borderRadius:20,padding:“8px 8px”,
display:“flex”,alignItems:“center”,justifyContent:“space-around”,
backdropFilter:“blur(20px)”
}}>
{tabs.map(tab=>{
const isActive = active===tab.id;
return (
<button key={tab.id} onClick={()=>onChange(tab.id)}
style={{
flex:1,padding:“8px 4px”,borderRadius:14,border:“none”,cursor:“pointer”,
background: isActive ? “#00ff88” : “transparent”,
transition:“all .2s”,display:“flex”,flexDirection:“column”,alignItems:“center”,gap:3
}}>
<span style={{fontSize:17}}>{tab.icon}</span>
<span style={{fontSize:10,fontWeight:700,fontFamily:”‘Syne’,sans-serif”,color: isActive ? “#030712” : “rgba(255,255,255,.35)”}}>
{tab.label}
</span>
</button>
);
})}
</div>
</div>
);
}

// ── Lang Toggle ────────────────────────────────────────────────────────────
function LangToggle({lang, setLang}) {
const next = lang===“en”?“ar”:“en”;
return (
<button onClick={()=>{setLang(next);document.documentElement.dir=next===“ar”?“rtl”:“ltr”;}}
style={{
padding:“5px 10px”,borderRadius:10,background:“rgba(255,255,255,.07)”,
border:“1px solid rgba(255,255,255,.1)”,color:“rgba(255,255,255,.5)”,
fontSize:11,fontWeight:700,cursor:“pointer”,fontFamily:”‘Syne’,sans-serif”
}}>
{next===“ar”?“العربية”:“English”}
</button>
);
}

// ── Root App ───────────────────────────────────────────────────────────────
function App() {
const [lang, setLang]     = useState(“en”);
const t = T[lang];
const [tab, setTab]       = useState(“home”);
const [user, setUser]     = useState(null);
const [deals, setDeals]   = useState([]);
const [loading, setLoading] = useState(false);
const [showCreate, setShowCreate] = useState(false);

const tgUserId = window.Telegram?.WebApp?.initDataUnsafe?.user?.id || null;

// Init Telegram WebApp
useEffect(() => {
const tg = window.Telegram?.WebApp;
if (tg) {
tg.ready(); tg.expand();
try { tg.setHeaderColor(”#030712”); tg.setBackgroundColor(”#030712”); } catch {}
const u = tg.initDataUnsafe?.user;
if (u) {
setUser(u);
if (u.language_code===“ar”) { setLang(“ar”); document.documentElement.dir=“rtl”; }
}
// Deep link: ?deal=UUID
const params = new URLSearchParams(window.location.search);
const dealId = params.get(“deal”);
if (dealId) { setTab(“trades”); loadDeals(); }
const page = params.get(“page”);
if (page===“create”) setShowCreate(true);
}
loadDeals();
}, []);

const loadDeals = async () => {
setLoading(true);
try {
const data = await apiFetch(”/deals/my”);
setDeals(data);
} catch(e) { toastErr(e.message||t.error); }
finally { setLoading(false); }
};

const addDeal = d => setDeals(prev=>[d,…prev]);
const updateDeal = d => setDeals(prev=>prev.map(x=>x.deal_uid===d.deal_uid?d:x));

const dir = t.dir;

return (
<div style={{minHeight:“100vh”,background:”#030712”,color:”#fff”,fontFamily:”‘Syne’,sans-serif”}} dir={dir}>
{/* Top bar */}
<div style={{
position:“fixed”,top:0,left:0,right:0,zIndex:20,
padding:“10px 16px”,display:“flex”,justifyContent:“flex-end”,
background:“linear-gradient(to bottom, rgba(3,7,18,.8) 0%, transparent 100%)”
}}>
<LangToggle lang={lang} setLang={setLang}/>
</div>

```
  {/* Page content */}
  <div style={{paddingTop:52}}>
    {tab==="home"   && <HomeTab   t={t} user={user} deals={deals} onOpenCreate={()=>setShowCreate(true)} onSwitchTab={setTab}/>}
    {tab==="trades" && <TradesTab t={t} deals={deals} tgUserId={tgUserId} onUpdate={updateDeal} onRefresh={loadDeals} loading={loading}/>}
    {tab==="wallet" && <WalletTab t={t} user={user} deals={deals} tgUserId={tgUserId}/>}
  </div>

  <BottomNav t={t} active={tab} onChange={setTab}/>

  {showCreate && (
    <CreateDealModal
      t={t}
      onClose={()=>setShowCreate(false)}
      onCreated={addDeal}
    />
  )}

  <Toast/>
</div>
```

);
}

ReactDOM.createRoot(document.getElementById(“root”)).render(<App/>);
</script>

</body>
</html>"""

# —————————————————————————

# FastAPI App

# —————————————————————————

bot, dp = None, None

@asynccontextmanager
async def lifespan(app: FastAPI):
global bot, dp
log.info(“Starting EscrowBot…”)
await create_tables()

```
if BOT_TOKEN:
    bot, dp = setup_bot()
    if ENVIRONMENT == "production" and WEBHOOK_URL:
        await bot.set_webhook(f"{WEBHOOK_URL}/webhook")
        log.info(f"Webhook set: {WEBHOOK_URL}/webhook")
    else:
        log.info("Starting bot polling (dev mode)")
        asyncio.create_task(dp.start_polling(bot))
else:
    log.warning("BOT_TOKEN not set - bot disabled")

yield

if bot:
    if ENVIRONMENT == "production":
        await bot.delete_webhook()
    await bot.session.close()
log.info("Shutdown complete")
```

app = FastAPI(title=“EscrowBot”, lifespan=lifespan)

app.add_middleware(
CORSMiddleware,
allow_origins=[”*”],
allow_credentials=True,
allow_methods=[”*”],
allow_headers=[”*”],
)

# ── Serve the Mini App ──────────────────────────────────────────────────────

@app.get(”/”, response_class=HTMLResponse)
async def serve_app():
html = HTML_APP.replace(“window.**BOT_USERNAME** || "EscrowBot"”, f”"{BOT_USERNAME}"”)
return HTMLResponse(content=html)

# ── Health ──────────────────────────────────────────────────────────────────

@app.get(”/health”)
async def health():
return {“status”: “ok”, “environment”: ENVIRONMENT}

# ── Webhook ─────────────────────────────────────────────────────────────────

@app.post(”/webhook”)
async def webhook(request: Request):
if not bot or not dp:
return JSONResponse({“ok”: False, “error”: “Bot not configured”}, status_code=200)
try:
data = await request.json()
update = Update(**data)
await dp.feed_update(bot, update)
return JSONResponse({“ok”: True})
except Exception as e:
log.error(f”Webhook error: {e}”)
return JSONResponse({“ok”: False}, status_code=200)

# ── Auth dependency ─────────────────────────────────────────────────────────

async def get_tg_user(x_init_data: str = Header(””)):
# In development/testing, allow empty init data with a mock user
if not x_init_data:
if ENVIRONMENT != “production”:
return {“id”: 999999, “first_name”: “TestUser”, “language_code”: “en”}
raise HTTPException(status_code=401, detail=“Missing init data”)
user_data = verify_telegram_init_data(x_init_data)
if not user_data:
if ENVIRONMENT != “production”:
return {“id”: 999999, “first_name”: “TestUser”, “language_code”: “en”}
raise HTTPException(status_code=401, detail=“Invalid Telegram init data”)
return user_data

# ── API Routes ──────────────────────────────────────────────────────────────

@app.get(”/api/fee-info”)
async def fee_info(amount: float):
if amount <= 0:
raise HTTPException(status_code=400, detail=“Amount must be positive”)
fee, fee_type = calculate_fee(amount)
return {
“amount”:     amount,
“fee”:        fee,
“fee_type”:   fee_type,
“total”:      round(amount + fee, 2),
“percentage”: FEE_PERCENT if fee_type == “percentage” else None,
}

@app.post(”/api/deals”)
async def create_deal(body: DealCreate, user_data: dict = Depends(get_tg_user)):
async with AsyncSessionLocal() as db:
user = await get_or_create_user(db, int(user_data[“id”]), user_data)
fee, _ = calculate_fee(body.amount)
deal = Deal(
seller_id=user.telegram_id,
title=body.title,
description=body.description,
amount=body.amount,
fee=fee,
currency=body.currency,
)
db.add(deal)
await db.commit()
await db.refresh(deal)
return deal_to_dict(deal)

@app.get(”/api/deals/my”)
async def my_deals(user_data: dict = Depends(get_tg_user)):
async with AsyncSessionLocal() as db:
user = await get_or_create_user(db, int(user_data[“id”]), user_data)
tid = user.telegram_id
from sqlalchemy import or_
res = await db.execute(
select(Deal)
.where(or_(Deal.seller_id == tid, Deal.buyer_id == tid))
.order_by(Deal.created_at.desc())
)
return [deal_to_dict(d) for d in res.scalars().all()]

@app.get(”/api/deals/{deal_uid}”)
async def get_deal(deal_uid: str):
async with AsyncSessionLocal() as db:
res = await db.execute(select(Deal).where(Deal.deal_uid == deal_uid))
deal = res.scalar_one_or_none()
if not deal:
raise HTTPException(status_code=404, detail=“Deal not found”)
return deal_to_dict(deal)

@app.post(”/api/deals/{deal_uid}/fund”)
async def fund_deal(deal_uid: str, body: FundBody, user_data: dict = Depends(get_tg_user)):
async with AsyncSessionLocal() as db:
user = await get_or_create_user(db, int(user_data[“id”]), user_data)
res = await db.execute(select(Deal).where(Deal.deal_uid == deal_uid))
deal = res.scalar_one_or_none()
if not deal or deal.status != DealStatus.PENDING:
raise HTTPException(status_code=400, detail=“Cannot fund this deal”)
if deal.seller_id == user.telegram_id:
raise HTTPException(status_code=400, detail=“Seller cannot fund own deal”)
deal.status    = DealStatus.FUNDED
deal.buyer_id  = user.telegram_id
deal.funded_at = datetime.now(timezone.utc)
await db.commit()
await db.refresh(deal)
return deal_to_dict(deal)

@app.post(”/api/deals/{deal_uid}/deliver”)
async def deliver_deal(deal_uid: str, body: DeliverBody, user_data: dict = Depends(get_tg_user)):
async with AsyncSessionLocal() as db:
res = await db.execute(select(Deal).where(Deal.deal_uid == deal_uid))
deal = res.scalar_one_or_none()
if not deal or deal.status != DealStatus.FUNDED:
raise HTTPException(status_code=400, detail=“Cannot mark as delivered”)
if deal.seller_id != int(user_data[“id”]):
raise HTTPException(status_code=403, detail=“Only seller can mark as delivered”)
deal.status         = DealStatus.DELIVERED
deal.delivery_proof = body.proof
deal.delivered_at   = datetime.now(timezone.utc)
await db.commit()
await db.refresh(deal)
return deal_to_dict(deal)

@app.post(”/api/deals/{deal_uid}/confirm”)
async def confirm_deal(deal_uid: str, user_data: dict = Depends(get_tg_user)):
async with AsyncSessionLocal() as db:
res = await db.execute(select(Deal).where(Deal.deal_uid == deal_uid))
deal = res.scalar_one_or_none()
if not deal or deal.status != DealStatus.DELIVERED:
raise HTTPException(status_code=400, detail=“Cannot confirm this deal”)
if deal.buyer_id != int(user_data[“id”]):
raise HTTPException(status_code=403, detail=“Only buyer can confirm receipt”)
deal.status       = DealStatus.COMPLETED
deal.completed_at = datetime.now(timezone.utc)
# Update seller trade count
s_res = await db.execute(select(User).where(User.telegram_id == deal.seller_id))
seller = s_res.scalar_one_or_none()
if seller:
seller.total_trades += 1
await db.commit()
await db.refresh(deal)
return deal_to_dict(deal)

@app.post(”/api/deals/{deal_uid}/dispute”)
async def dispute_deal(deal_uid: str, body: DisputeBody, user_data: dict = Depends(get_tg_user)):
async with AsyncSessionLocal() as db:
res = await db.execute(select(Deal).where(Deal.deal_uid == deal_uid))
deal = res.scalar_one_or_none()
if not deal or deal.status not in [DealStatus.FUNDED, DealStatus.DELIVERED]:
raise HTTPException(status_code=400, detail=“Cannot dispute this deal”)
uid = int(user_data[“id”])
if deal.buyer_id != uid and deal.seller_id != uid:
raise HTTPException(status_code=403, detail=“Not a party to this deal”)
deal.status         = DealStatus.DISPUTED
deal.dispute_reason = body.reason
await db.commit()
await db.refresh(deal)
return deal_to_dict(deal)

# —————————————————————————

# Entry point

# —————————————————————————

if **name** == “**main**”:
import uvicorn
uvicorn.run(“main:app”, host=“0.0.0.0”, port=8000, reload=True)
