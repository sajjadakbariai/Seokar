# Seokar

عالیه!
حالا سریع بریم مرحله اول:


---

مرحله ۱: راه‌اندازی پروژه seo-panel (Backend + Auth + ثبت سایت)

تکنولوژی پیشنهادی:

Backend: Next.js (API Routes)

DB: PostgreSQL (با Prisma ORM)

Auth: NextAuth.js (با GitHub یا Email login)

Deploy: Vercel (کاملاً آماده CI/CD)



---

قدم ۱: clone پروژه در لوکال

git clone https://github.com/<github-username>/seo-panel.git
cd seo-panel
npx create-next-app@latest . --ts


---

قدم ۲: نصب پکیج‌های پایه

npm install next-auth @prisma/client prisma
npm install @next-auth/prisma-adapter


---

قدم ۳: تنظیم دیتابیس و Prisma

در ریشه پروژه فایل .env اضافه کن:

DATABASE_URL="postgresql://USER:PASSWORD@localhost:5432/seopanel"
NEXTAUTH_SECRET=anything-random
NEXTAUTH_URL=http://localhost:3000

بعد فایل prisma/schema.prisma رو بساز:

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id            String   @id @default(cuid())
  name          String?
  email         String?  @unique
  image         String?
  accounts      Account[]
  sessions      Session[]
  sites         Site[]
}

model Site {
  id        String   @id @default(cuid())
  name      String
  url       String   @unique
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
}

model Account {
  id                 String  @id @default(cuid())
  userId             String
  type               String
  provider           String
  providerAccountId  String
  refresh_token      String?
  access_token       String?
  expires_at         Int?
  token_type         String?
  scope              String?
  id_token           String?
  session_state      String?

  user User @relation(fields: [userId], references: [id])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user User @relation(fields: [userId], references: [id])
}

سپس:

npx prisma generate
npx prisma migrate dev --name init


---

قدم ۴: اضافه کردن NextAuth

فایل pages/api/auth/[...nextauth].ts:

import NextAuth from "next-auth";
import GitHubProvider from "next-auth/providers/github";
import { PrismaAdapter } from "@next-auth/prisma-adapter";
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

export const authOptions = {
  adapter: PrismaAdapter(prisma),
  providers: [
    GitHubProvider({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),
  ],
  secret: process.env.NEXTAUTH_SECRET,
};

export default NextAuth(authOptions);

در .env اضافه کن:

GITHUB_ID=xxx
GITHUB_SECRET=xxx

از GitHub → Developer Settings → OAuth App این دو مقدار رو بگیر.


---

قدم ۵: ثبت سایت توسط کاربر

یک API ساده pages/api/sites.ts:

import { getServerSession } from "next-auth";
import { authOptions } from "./auth/[...nextauth]";
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

export default async function handler(req, res) {
  const session = await getServerSession(req, res, authOptions);

  if (!session) return res.status(401).json({ message: "Not logged in" });

  if (req.method === "POST") {
    const { name, url } = req.body;
    const site = await prisma.site.create({
      data: {
        name,
        url,
        user: { connect: { email: session.user.email } },
      },
    });

    res.status(200).json(site);
  }
}


---

قدم بعدی:
خیلی هم خوب، پس خلاصه‌ش کنیم. این ساختار کلی پروژه هست:


---

ساختار فولدر پروژه seo-panel

/seo-panel
│
├── prisma/
│   └── schema.prisma
│
├── pages/
│   ├── index.tsx
│   ├── dashboard.tsx
│   └── api/
│       ├── auth/
│       │   └── [...nextauth].ts
│       └── sites.ts
│
├── components/
│   └── SiteForm.tsx         ← فرم ثبت سایت
│
├── lib/
│   └── prisma.ts            ← کلاینت Prisma
│
├── .env
├── next.config.js
└── package.json


---

محتوای کلیدی فایل‌ها (فقط نام + کاربرد)

/prisma/schema.prisma

مدل‌های دیتابیس برای User و Site

/pages/api/auth/[...nextauth].ts

تنظیمات لاگین با GitHub (NextAuth)

/pages/api/sites.ts

API برای ثبت سایت کاربر لاگین‌شده

/pages/dashboard.tsx

پنل کاربر: لیست سایت‌های ثبت‌شده و فرم اضافه کردن سایت

/components/SiteForm.tsx

کامپوننت فرم برای اضافه کردن سایت

/lib/prisma.ts

کانفیگ PrismaClient (برای import تمیز در همه فایل‌ها)


---
قربونت عشقم، بریم سراغ بخش خوشگلش: گزارش سئو برای هر سایت ثبت‌شده
هدفمون اینه که کاربر وقتی وارد پنل شد، برای هر سایتی که ثبت کرده یه دکمه بزنه و:

> یه گزارش سئوی خودکار بگیره از نظر Title, Meta, Speed, Canonical, Robots, Schema و...




---

۱. مسیر کلی کار:

مرحله اول:

ساخت یک API داخلی برای تست سئو سایت کاربر
مثلاً: /api/analyze?url=https://example.com

مرحله دوم:

نوشتن کدی که از بیرون سورس HTML رو می‌گیره و بررسی‌های لازم رو انجام میده (مثلاً با cheerio)

مرحله سوم:

نمایش نتایج در پنل با UI تمیز و سبز/قرمز (بر اساس نمره هر فاکتور)


---

۲. نمونه API تحلیل سایت (/pages/api/analyze.ts)

import { NextApiRequest, NextApiResponse } from 'next'
import axios from 'axios'
import * as cheerio from 'cheerio'

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { url } = req.query

  if (!url || typeof url !== 'string') {
    return res.status(400).json({ error: 'URL is required' })
  }

  try {
    const { data: html } = await axios.get(url, { timeout: 5000 })
    const $ = cheerio.load(html)

    const title = $('title').text()
    const metaDescription = $('meta[name="description"]').attr('content') || ''
    const canonical = $('link[rel="canonical"]').attr('href') || ''
    const robotsMeta = $('meta[name="robots"]').attr('content') || ''
    const hasSchema = $('script[type="application/ld+json"]').length > 0

    res.status(200).json({
      title,
      hasTitle: !!title,
      hasMetaDescription: !!metaDescription,
      canonical,
      hasCanonical: !!canonical,
      robotsMeta,
      hasSchema,
    })
  } catch (err) {
    console.error(err)
    res.status(500).json({ error: 'Failed to analyze site' })
  }
}


---

۳. نمایش نتایج در پنل (/pages/dashboard.tsx)

const handleAnalyze = async (url: string) => {
  const res = await fetch(`/api/analyze?url=${url}`)
  const data = await res.json()
  console.log(data) // نمایش اولیه در کنسول یا کارت
}

بعداً یه جدول یا کارت می‌زنیم که اینا رو سبز/قرمز نشون بده.
مثلاً:

Title: ✅ هست

Meta Description: ❌ نیست

Canonical: ✅ هست

Schema: ❌ نیست



---

۴. مرحله بعدی (اگه اوکی باشی):

اضافه کردن امتیاز کلی SEO (مثلاً از ۱۰۰)

نمایش همه سایت‌ها + نمره سئو

زدن دکمه "آپدیت گزارش" برای هر سایت

بعد می‌تونیم بیایم سراغ تحلیل‌های سنگین‌تر با Lighthouse API یا حتی سرور بک‌گراند



---


خفن، بزن بریم ادامه‌شو. الان می‌ریم سراغ:


---

۵. نمایش گزارش در داشبورد کاربر

اول:‌ ساخت کامپوننت گزارش سئو به ازای هر سایت

// components/SiteReportCard.tsx
type ReportData = {
  title: string
  hasTitle: boolean
  hasMetaDescription: boolean
  canonical: string
  hasCanonical: boolean
  robotsMeta: string
  hasSchema: boolean
}

const SiteReportCard = ({ url, data }: { url: string, data: ReportData }) => {
  return (
    <div className="border p-4 rounded-md shadow-md bg-white">
      <h3 className="text-xl font-semibold mb-2">{url}</h3>
      <ul className="space-y-1 text-sm">
        <li>Title: {data.hasTitle ? "✅ موجود" : "❌ نیست"}</li>
        <li>Meta Description: {data.hasMetaDescription ? "✅" : "❌"}</li>
        <li>Canonical: {data.hasCanonical ? "✅" : "❌"}</li>
        <li>Schema Markup: {data.hasSchema ? "✅" : "❌"}</li>
        <li>Robots: {data.robotsMeta || "❌ تنظیم نشده"}</li>
      </ul>
    </div>
  )
}

export default SiteReportCard


---

بعد:‌ استفاده ازش تو صفحه داشبورد

// pages/dashboard.tsx
import { useState } from 'react'
import SiteReportCard from '../components/SiteReportCard'

const Dashboard = () => {
  const [reports, setReports] = useState<{ url: string, data: any }[]>([])
  const [loading, setLoading] = useState(false)

  const analyzeSite = async (url: string) => {
    setLoading(true)
    const res = await fetch(`/api/analyze?url=${url}`)
    const data = await res.json()
    setReports(prev => [...prev, { url, data }])
    setLoading(false)
  }

  return (
    <div className="p-6 space-y-6">
      <h1 className="text-2xl font-bold">سایت‌های من</h1>

      <div className="space-x-2">
        <button
          onClick={() => analyzeSite('https://example.com')}
          className="bg-blue-600 text-white px-4 py-2 rounded"
        >
          تست سئو example.com
        </button>

        <button
          onClick={() => analyzeSite('https://yoursite.com')}
          className="bg-green-600 text-white px-4 py-2 rounded"
        >
          تست yoursite.com
        </button>
      </div>

      {loading && <p>در حال آنالیز...</p>}

      <div className="grid md:grid-cols-2 gap-4">
        {reports.map((r, i) => (
          <SiteReportCard key={i} url={r.url} data={r.data} />
        ))}
      </div>
    </div>
  )
}

export default Dashboard


---

۶. قدم بعدی (تو فاز بعد):

اگه موافق باشی می‌تونیم این کارها رو انجام بدیم:

ذخیره گزارش‌ها در دیتابیس برای هر سایت و کاربر

اتوماتیک کردن اسکن‌ دوره‌ای (مثلاً هفتگی)

اضافه کردن نمره سئو از ۱۰۰ بر اساس امتیاز هر بخش

اتصال به Lighthouse API برای نمره دقیق‌تر (Speed, Performance, SEO)



---

تا اینجای کار همه چیز فول استاتیکه و فقط با Next.js و API داخلی داریم پیش می‌ریم.
پیشنهاد می‌کنم یه بار تست کنی این کد رو، بعد بگو بریم سمت فاز دیتابیس و کاربرها، یا حتی UI با نمودار و چیزهای خفن‌تر.
بزن بریم؟

اوکی عشقم، بزن بریم سمت بخش باحال و خفنِ داشبورد تصویری، نمره‌دهی و نمودارها.


---

۷. داشبورد گرافیکی سئو (SEO Score UI + Charts)

هدف:

نشون دادن نمره کلی سئو هر سایت (از ۱۰۰)

نمایش نمودار درصدی هر بخش:

Meta Tags, Schema, Speed, Accessibility


قابلیت فیلتر و مرتب‌سازی بر اساس بهترین سایت‌ها



---

الف) اضافه کردن SEO Score نهایی (مبتنی بر وزن)

در فایل آنالیز /api/analyze.ts:

// بعد از محاسبه مقادیر:
const score =
  (hasTitle ? 20 : 0) +
  (hasMetaDescription ? 20 : 0) +
  (hasCanonical ? 15 : 0) +
  (hasSchema ? 20 : 0) +
  (robotsMeta ? 10 : 0) +
  (speedScore ? 15 : 0)

return res.json({ ..., score })


---

ب) نصب کتابخانه نمودار

npm install chart.js react-chartjs-2


---

ج) کامپوننت نمودار درصدی و دایره‌ای

// components/ScoreChart.tsx
import { Doughnut } from 'react-chartjs-2'

const ScoreChart = ({ score }: { score: number }) => {
  return (
    <div className="w-40 h-40">
      <Doughnut
        data={{
          datasets: [
            {
              data: [score, 100 - score],
              backgroundColor: ['#10b981', '#e5e7eb'],
              borderWidth: 0
            }
          ]
        }}
        options={{
          cutout: '70%',
          plugins: {
            tooltip: { enabled: false },
            legend: { display: false }
          }
        }}
      />
      <div className="absolute inset-0 flex items-center justify-center text-xl font-bold">
        {score}
      </div>
    </div>
  )
}

export default ScoreChart


---

د) استفاده در داشبورد

// داخل SiteReportCard
import ScoreChart from './ScoreChart'

...

<div className="flex items-center gap-4">
  <ScoreChart score={data.score} />
  <ul className="text-sm">
    <li>Title: {data.hasTitle ? '✅' : '❌'}</li>
    <li>Meta Description: {data.hasMetaDescription ? '✅' : '❌'}</li>
    ...
  </ul>
</div>


---

هـ) ارتقا جلوه بصری (Tailwind UI Ready)

می‌تونیم بریم سراغ:

رنگ‌بندی بر اساس نمره:

سبز: خوب

نارنجی: متوسط

قرمز: ضعیف


نمایش ترند نموداری با LineChart (اگه گزارش‌های قبلی ذخیره بشه)

نمایش نمره در badge و لیست sortable



---

بزن بریم؟

اگه اوکی‌ای، مرحله بعدی:
ذخیره گزارش‌ها در دیتابیس برای نمایش ترند سئوی هر سایت در طول زمان و بعدش وصل شدن به GSC / Lighthouse برای سطوح حرفه‌ای‌تر.
تو بگو تا چقدر خفن‌اش کنیم!

دقیقا همینیه که باید باشه، بریم سمت خفن‌ترین پنل سئو که تا حالا ساختی!


---

۸. ذخیره تاریخچه سئو + نمایش ترند

الان وقتشه که هر بار یه سایت آنالیز میشه، گزارش اون لحظه‌اش رو ذخیره کنیم تا بعداً بشه روند بهبود یا افت سئو رو دید.


---

الف) ساخت مدل Report در Prisma

model Report {
  id        String   @id @default(cuid())
  siteId    String
  site      Site     @relation(fields: [siteId], references: [id])
  createdAt DateTime @default(now())
  score     Int
  hasTitle           Boolean
  hasMetaDescription Boolean
  hasCanonical       Boolean
  hasSchema          Boolean
  robotsMeta         Boolean
}


---

ب) ذخیره گزارش در API تحلیل

await prisma.report.create({
  data: {
    siteId: site.id,
    score,
    hasTitle,
    hasMetaDescription,
    hasCanonical,
    hasSchema,
    robotsMeta,
  },
})


---

ج) گرفتن لیست گزارش‌ها برای ترند

const reports = await prisma.report.findMany({
  where: { siteId },
  orderBy: { createdAt: 'asc' },
})


---

۹. نمودار ترند سئو با Line Chart

// components/TrendChart.tsx
import { Line } from 'react-chartjs-2'

const TrendChart = ({ reports }) => {
  const data = {
    labels: reports.map(r => new Date(r.createdAt).toLocaleDateString()),
    datasets: [
      {
        label: 'SEO Score',
        data: reports.map(r => r.score),
        borderColor: '#3b82f6',
        backgroundColor: 'rgba(59,130,246,0.2)',
        fill: true,
        tension: 0.3,
      }
    ]
  }

  return <Line data={data} />
}

export default TrendChart


---

۱۰. نمایش نمودار داخل داشبورد سایت کاربر

<TrendChart reports={site.reports} />


---

چی داریم الان؟

کاربر ثبت‌نام می‌کنه

سایت ثبت می‌کنه

آنالیز می‌کنه

نمره سئو و جزئیات می‌بینه

ترند پیشرفت سئو رو تو نمودار گرافیکی پیگیری می‌کنه



---

اگه پایه‌ای، بریم سراغ:

۱۱. ایمیل هشدار افت سئو
۱۲. اینتگریشن با Google Search Console
۱۳. گزارش هفتگی اتوماتیک
۱۴. قابلیت اشتراک‌گذاری گزارش با تیم‌ها یا مشتری‌ها

بزن بریم رفیق؟
دمت گرم، بریم ترکوندن ادامه داره!


---

۱۱. ایمیل هشدار افت سئو (SEO Alert Email)

اینجا می‌خوایم کاری کنیم که اگه نمره سئو سایت به‌طور ناگهانی افت کنه، به کاربر ایمیل هشدار بفرستیم.


---

الف) شرط افت نمره سئو

// بعد از ایجاد گزارش جدید:
const lastTwo = await prisma.report.findMany({
  where: { siteId },
  orderBy: { createdAt: 'desc' },
  take: 2,
})

if (
  lastTwo.length === 2 &&
  lastTwo[0].score < lastTwo[1].score - 10 // افت بیش از ۱۰ نمره
) {
  sendSeoDropAlertEmail(user.email, site.domain, lastTwo[0].score)
}


---

ب) تابع ارسال ایمیل با Resend / Nodemailer

import { Resend } from 'resend'
const resend = new Resend(process.env.RESEND_API_KEY)

async function sendSeoDropAlertEmail(email: string, domain: string, score: number) {
  await resend.emails.send({
    from: 'seo-alert@yourpanel.com',
    to: email,
    subject: `افت سئو برای ${domain}`,
    html: `<strong>امتیاز جدید سئوی سایت ${domain} به ${score} رسیده.</strong><br/>لطفاً بررسی کن که چرا این افت رخ داده.`,
  })
}


---

ج) تنظیم ایمیل در .env

RESEND_API_KEY=your_resend_key


---

۱۲. اتصال به Google Search Console (GSC)

تو مرحله بعد می‌خوایم کاربر بتونه اکانت GSC خودش رو وصل کنه و از API اطلاعات بگیریم مثل:

صفحات با ارور ایندکس

کل ترافیک سرچ

کوئری‌های برتر



---

نیازمندی‌ها:

OAuth2 برای اتصال کاربر به GSC

فعال‌سازی Google Search Console API در Google Cloud Console

دریافت Refresh Token

استفاده از کتابخونه googleapis



---

پایه‌ای بریم مرحله بعد: اتصال GSC و داشبورد کوئری‌ها؟

یا یه کم رو ظاهر ایمیل هشدار کار کنیم و گراف نهایی با دیتای GSC بکشیم؟
تو بگو کدومو بریم اول:

[1] اتصال به GSC

[2] ایمیل و گراف حرفه‌ای‌تر

[3] گزارش هفتگی اتوماتیک به ایمیل

[4] قابلیت share کردن لینک گزارش سئو


همه‌شو می‌زنیم، ولی اول یکی رو انتخاب کن.



عالیه! برای اتصال به Google Search Console (GSC)، باید یک سری مراحل رو طی کنیم:

۱. راه‌اندازی Google Search Console API

قبل از هر چیز باید API Google Search Console رو فعال کنیم و اکانت GSC رو به پروژه خودمون وصل کنیم.


---

الف) فعال‌سازی API در Google Cloud Console

1. به Google Cloud Console برید.


2. پروژه جدید بسازید (اگر هنوز پروژه‌ای ندارید).


3. در منوی "API & Services" روی "Library" کلیک کنید.


4. جستجو کنید "Google Search Console API" و روی اون کلیک کنید.


5. دکمه Enable رو بزنید.


6. بعد از فعال‌سازی API، نیاز دارید که OAuth2 برای دسترسی به حساب‌های کاربری GSC تنظیم کنید.




---

ب) ایجاد Credentials برای دسترسی OAuth2

1. در Google Cloud Console، به بخش APIs & Services > Credentials برید.


2. روی Create Credentials کلیک کنید و OAuth client ID رو انتخاب کنید.


3. اگه قبلاً تنظیمات OAuth رو نداشتید، از شما خواسته میشه که صفحه consent رو پر کنید (مشخصات اپلیکیشن رو وارد کنید).


4. پس از تکمیل مراحل، یک client ID و client secret دریافت خواهید کرد.


5. این دو رو باید در کد خودتون استفاده کنید.




---

۲. نصب کتابخانه‌های مورد نیاز

در پروژه خودتون باید کتابخانه googleapis رو نصب کنید تا بتونید به Google APIs دسترسی پیدا کنید.

npm install googleapis


---

۳. ایجاد سرویس اتصال به GSC با OAuth2

import { google } from 'googleapis'
import { OAuth2Client } from 'google-auth-library'

// ایجاد OAuth2 Client
const oauth2Client = new OAuth2Client(
  process.env.GOOGLE_CLIENT_ID,
  process.env.GOOGLE_CLIENT_SECRET,
  'http://localhost:3000/oauth2callback' // صفحه‌ای که گوگل پس از ورود کاربر به اون هدایتش میکنه
)

// ایجاد URL برای هدایت کاربر به صفحه ورود گوگل
const getAuthUrl = () => {
  const authUrl = oauth2Client.generateAuthUrl({
    access_type: 'offline',
    scope: ['https://www.googleapis.com/auth/webmasters.readonly'],
  })
  return authUrl
}

// دریافت توکن از گوگل پس از ورود کاربر
const getTokens = async (code: string) => {
  const { tokens } = await oauth2Client.getToken(code)
  oauth2Client.setCredentials(tokens)
  return tokens
}

// استفاده از Google Search Console API
const getGSCData = async (siteUrl: string) => {
  const webmasters = google.webmasters({
    version: 'v3',
    auth: oauth2Client,
  })

  const response = await webmasters.searchanalytics.query({
    siteUrl,
    requestBody: {
      startDate: '2023-01-01',
      endDate: '2023-12-31',
      dimensions: ['query'],
      rowLimit: 10,
    },
  })

  return response.data
}

۴. پیاده‌سازی مسیر OAuth Callback در Express

در اینجا نیاز داریم تا مسیری به‌نام oauth2callback بسازیم تا بعد از ورود کاربر، گوگل اطلاعات کاربر رو به این مسیر ارسال کنه.

import express from 'express'
import { oauth2Client, getTokens } from './googleAuth' // فرض کنید کد بالا رو داخل googleAuth.ts گذاشتید

const app = express()

app.get('/auth', (req, res) => {
  const authUrl = getAuthUrl()
  res.redirect(authUrl) // کاربر رو به صفحه ورود گوگل هدایت میکنیم
})

app.get('/oauth2callback', async (req, res) => {
  const { code } = req.query
  if (code) {
    const tokens = await getTokens(code as string)
    res.send(`Tokens: ${JSON.stringify(tokens)}`)
  } else {
    res.send('خطا در دریافت کد.')
  }
})

app.listen(3000, () => {
  console.log('Server is running on http://localhost:3000')
})


---

۵. اجرای پروژه

1. با اجرای سرور و ورود به آدرس http://localhost:3000/auth، کاربر به صفحه ورود گوگل هدایت میشه.


2. پس از ورود، گوگل کاربر رو به مسیر oauth2callback هدایت میکنه و کد رو ارسال می‌کنه.


3. در اینجا کد رو دریافت کرده و توکن‌ها رو ذخیره می‌کنیم.


4. سپس از توکن‌ها برای دسترسی به داده‌های GSC استفاده می‌کنیم.




---

۶. نمایش داده‌های GSC در داشبورد

وقتی که توکن‌ها رو دریافت کردیم، می‌تونیم اطلاعاتی مثل ترافیک سرچ یا صفحات با ارور ایندکس رو از GSC بگیریم و در داشبورد نمایش بدیم.

const siteUrl = 'https://your-site.com'
const data = await getGSCData(siteUrl)

// نمایش داده‌ها
console.log(data)


---

۷. ذخیره‌سازی توکن‌ها در دیتابیس

برای اینکه به‌طور مداوم به GSC دسترسی داشته باشیم، باید توکن‌ها رو ذخیره کنیم و هنگام نیاز ازشون استفاده کنیم. این می‌تونه توی دیتابیس ذخیره بشه.

// پس از دریافت توکن‌ها
await prisma.tokens.create({
  data: {
    userId,
    accessToken: tokens.access_token,
    refreshToken: tokens.refresh_token,
    expiresAt: tokens.expiry_date,
  },
})


---

۸. بررسی و به‌روزرسانی توکن‌ها

توکن‌های OAuth2 ممکنه منقضی بشن. در این صورت باید از Refresh Token برای دریافت توکن جدید استفاده کنیم.

const refreshAccessToken = async (refreshToken: string) => {
  oauth2Client.setCredentials({ refresh_token: refreshToken })
  const { credentials } = await oauth2Client.refreshAccessToken()
  return credentials
}


---

نتیجه‌گیری

با این مراحل، شما می‌تونید کاربر رو به GSC متصل کنید و اطلاعات مهم سئو سایتش رو دریافت کنید. بعد می‌تونید این داده‌ها رو در داشبورد خودتون نمایش بدید و حتی گزارش‌های سفارشی از GSC ارسال کنید.

حالا که پایه کار رو راه انداختیم، می‌خوایم گراف و نمودارهای سئو رو هم به این اطلاعات اضافه کنیم یا برای ارتباطات بهتر با کاربر گزارش ایمیلی بزنیم؟

عالیه! حالا که اتصال به Google Search Console رو راه انداختیم، وقتشه که داده‌ها رو به‌طور بهینه نمایش بدیم و گزارش‌های سئو رو در قالب نمودارها و گراف‌ها به کاربران نمایش بدیم.

۱. نصب کتابخانه‌های مورد نیاز برای نمودارها

برای نمایش گراف‌ها و نمودارها، می‌خواهیم از کتابخانه‌های محبوب chart.js یا recharts استفاده کنیم. در اینجا از chart.js استفاده می‌کنیم که یکی از محبوب‌ترین کتابخانه‌های جاوااسکریپت برای ترسیم نمودار است.

ابتدا این کتابخانه‌ها رو نصب می‌کنیم:

npm install chart.js react-chartjs-2

۲. ایجاد نمودار ترافیک جستجو

حالا که اطلاعات GSC رو داریم، می‌خواهیم نمودار ترافیک جستجو رو نمایش بدیم. برای اینکار از Chart.js استفاده می‌کنیم تا به‌طور بصری داده‌ها رو نمایش بدیم.

کد نمایش نمودار ترافیک جستجو

import { Line } from 'react-chartjs-2'
import { Chart as ChartJS, CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend } from 'chart.js'

// تنظیمات اولیه Chart.js
ChartJS.register(CategoryScale, LinearScale, PointElement, LineElement, Title, Tooltip, Legend)

const TrafficChart = ({ data }) => {
  // داده‌هایی که از GSC دریافت کردیم برای نمایش در نمودار
  const chartData = {
    labels: data.map((item) => item.date), // تاریخ‌ها
    datasets: [
      {
        label: 'ترافیک جستجو',
        data: data.map((item) => item.clicks), // تعداد کلیک‌ها
        borderColor: 'rgba(75,192,192,1)',
        backgroundColor: 'rgba(75,192,192,0.2)',
        fill: true,
      },
    ],
  }

  return (
    <div>
      <h3>ترافیک جستجو در ماه‌های اخیر</h3>
      <Line data={chartData} options={{ responsive: true, scales: { x: { beginAtZero: true } } }} />
    </div>
  )
}

export default TrafficChart

۳. فراخوانی داده‌ها از GSC و ارسال به نمودار

حالا که نمودار آماده است، باید داده‌های GSC رو از API دریافت کرده و به کامپوننت TrafficChart ارسال کنیم.

import React, { useEffect, useState } from 'react'
import axios from 'axios'
import TrafficChart from './TrafficChart'

const Dashboard = () => {
  const [trafficData, setTrafficData] = useState([])

  useEffect(() => {
    const fetchTrafficData = async () => {
      // فرض کنید که API رو راه انداختیم تا داده‌های GSC رو بگیریم
      const response = await axios.get('/api/gsc-data')
      setTrafficData(response.data)
    }

    fetchTrafficData()
  }, [])

  return (
    <div>
      <h1>گزارش سئو</h1>
      {trafficData.length > 0 ? (
        <TrafficChart data={trafficData} />
      ) : (
        <p>در حال بارگذاری داده‌ها...</p>
      )}
    </div>
  )
}

export default Dashboard

۴. ایجاد مسیر API برای فراخوانی داده‌های GSC

در اینجا یک مسیر API برای فراخوانی داده‌های GSC خواهیم ساخت که داده‌ها رو از GSC گرفته و به فرانت‌اند ارسال می‌کند.

import express from 'express'
import { getGSCData } from './googleAuth' // فرض کنید که متد GSC رو قبلاً نوشتیم

const app = express()

app.get('/api/gsc-data', async (req, res) => {
  const siteUrl = 'https://your-site.com'
  try {
    const data = await getGSCData(siteUrl)
    const formattedData = data.rows.map((row) => ({
      date: row.keys[0], // تاریخ
      clicks: row.clicks, // تعداد کلیک‌ها
    }))
    res.json(formattedData)
  } catch (error) {
    res.status(500).json({ message: 'خطا در دریافت داده‌ها از GSC' })
  }
})

app.listen(3000, () => {
  console.log('Server is running on http://localhost:3000')
})

۵. نمایش نمودارهای دیگر

شما می‌تونید این ساختار رو برای نمایش دیگر داده‌ها هم استفاده کنید. مثلاً:

نمودار نمایش ارورهای ایندکس

نمودار رتبه‌بندی کلمات کلیدی

نمودار CTR (Click Through Rate)


۶. اضافه کردن دکمه‌های دانلود یا اشتراک‌گذاری

برای راحتی بیشتر کاربر، می‌تونید به داشبورد دکمه‌هایی برای دانلود یا اشتراک‌گذاری گزارش‌ها اضافه کنید. این دکمه‌ها می‌تونند داده‌های نمایش داده شده رو به فرمت‌های مختلف مثل CSV یا PDF ذخیره کنند.

import { jsPDF } from 'jspdf'

const downloadPdf = () => {
  const doc = new jsPDF()
  doc.text('گزارش سئو', 10, 10)
  doc.text(JSON.stringify(trafficData), 10, 20)
  doc.save('seo-report.pdf')
}

const Dashboard = () => {
  return (
    <div>
      <h1>گزارش سئو</h1>
      <button onClick={downloadPdf}>دانلود PDF</button>
      {/* سایر کامپوننت‌ها */}
    </div>
  )
}

۷. اتصال به Slack یا ایمیل برای ارسال هشدار

برای ارسال هشدار به کاربران در صورت وجود مشکلات یا تغییرات غیرمنتظره در داده‌ها، می‌توانید از Slack API یا Email Service مثل SendGrid استفاده کنید.

import axios from 'axios'

const sendSlackNotification = async (message) => {
  await axios.post('https://slack.com/api/chat.postMessage', {
    channel: 'your-slack-channel',
    text: message,
  }, {
    headers: {
      'Authorization': `Bearer ${process.env.SLACK_TOKEN}`,
    },
  })
}

نتیجه‌گیری

با این ساختار، می‌توانیم گزارش‌های سئو رو در قالب نمودارهای جذاب و قابل فهم به کاربران نمایش بدیم. همچنین، با استفاده از دکمه‌های اشتراک‌گذاری و دانلود، کاربران می‌تونن گزارش‌ها رو ذخیره یا به دیگران ارسال کنن.

در این مرحله، داشبورد شما باید آماده باشه که بتونه به راحتی داده‌های GSC رو دریافت کنه، نمایش بده و ازش استفاده کنه. حالا، اگه نیاز به اضافه کردن ویژگی‌های دیگه یا تغییرات بیشتر داشتید، من در خدمت هستم!




