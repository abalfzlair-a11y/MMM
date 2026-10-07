# MarineForce — راهنمای استقرار

## ۱. استقرار روی Netlify

### مراحل:
1. پروژه را دانلود و از حالت فشرده خارج کنید
2. در سایت [netlify.com](https://netlify.com) حساب بسازید
3. روی «Add new site → Deploy manually» کلیک کنید
4. پوشه پروژه را Drag & Drop کنید

### یا با Git:
1. پروژه را به GitHub/GitLab push کنید
2. در Netlify روی «Add new site → Import from Git» کلیک کنید
3. ریپازیتوری را انتخاب کنید

### تنظیمات Netlify:
- **Build command:** `npm run build`
- **Publish directory:** `.next/standalone`
- **Node version:** 18 یا بالاتر

### فایل‌های مهم:
- `netlify.toml` — تنظیمات Netlify (در پوشه اصلی)
- `next.config.ts` — تنظیمات Next.js با `output: 'standalone'`

---

## ۲. راه‌اندازی Supabase (آینده)

در حال حاضر اپلیکیشن از **localStorage** مرورگر استفاده می‌کند (بدون نیاز به سرور).
برای اتصال به Supabase در آینده:

### مرحله ۱: ساخت پروژه Supabase
1. به [supabase.com](https://supabase.com) بروید
2. پروژه جدید بسازید
3. URL و Anon Key را از Settings > API کپی کنید

### مرحله ۲: نصب Supabase Client
```bash
npm install @supabase/supabase-js
```

### مرحله ۳: فایل `.env.local`
```
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx
```

### مرحله ۴: ساخت جداول در Supabase
در SQL Editor این اسکریپت را اجرا کنید:

```sql
-- جدول کاربران
CREATE TABLE users (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  role TEXT NOT NULL DEFAULT 'read_only',
  status TEXT DEFAULT 'active',
  password TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- جدول مشتریان
CREATE TABLE customers (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  customer_id TEXT UNIQUE,
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  company TEXT,
  email TEXT,
  phone TEXT,
  type TEXT DEFAULT 'Individual',
  status TEXT DEFAULT 'active',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- جدول شناورها
CREATE TABLE vessels (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  vessel_id TEXT UNIQUE,
  name TEXT NOT NULL,
  type TEXT,
  owner_id UUID REFERENCES customers(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- جدول موتورها
CREATE TABLE engines (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  engine_id TEXT UNIQUE,
  manufacturer TEXT,
  model TEXT,
  serial_number TEXT,
  vessel_id UUID REFERENCES vessels(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- جدول سفارشات کار
CREATE TABLE work_orders (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  work_order_number TEXT UNIQUE,
  customer_id UUID REFERENCES customers(id),
  vessel_id UUID REFERENCES vessels(id),
  complaint TEXT,
  status TEXT DEFAULT 'Open',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- جدول انبار
CREATE TABLE parts (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  sku TEXT UNIQUE,
  name TEXT NOT NULL,
  category TEXT,
  quantity INTEGER DEFAULT 0,
  min_quantity INTEGER DEFAULT 1,
  cost_price INTEGER DEFAULT 0,
  selling_price INTEGER DEFAULT 0,
  owner_type TEXT DEFAULT 'workshop'
);

-- جدول تأمین‌کنندگان
CREATE TABLE suppliers (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  company TEXT NOT NULL,
  contact_person TEXT,
  phone TEXT,
  email TEXT,
  outstanding_amount INTEGER DEFAULT 0,
  status TEXT DEFAULT 'active'
);

-- جدول فاکتورها
CREATE TABLE invoices (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  invoice_number TEXT UNIQUE,
  customer_id UUID REFERENCES customers(id),
  total INTEGER DEFAULT 0,
  paid INTEGER DEFAULT 0,
  remaining INTEGER DEFAULT 0,
  status TEXT DEFAULT 'Draft',
  issue_date DATE,
  due_date DATE
);

-- جدول پرداخت‌ها
CREATE TABLE payments (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  payment_number TEXT UNIQUE,
  invoice_id UUID REFERENCES invoices(id),
  amount INTEGER,
  method TEXT,
  date DATE
);

-- جدول چت
CREATE TABLE chat_conversations (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  created_by UUID,
  status TEXT DEFAULT 'active',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE chat_messages (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  conversation_id UUID REFERENCES chat_conversations(id),
  sender_id UUID,
  sender_name TEXT,
  text TEXT,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  deleted_by_user BOOLEAN DEFAULT FALSE
);

-- فعال‌سازی RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE vessels ENABLE ROW LEVEL SECURITY;
ALTER TABLE engines ENABLE ROW LEVEL SECURITY;
ALTER TABLE work_orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE parts ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_messages ENABLE ROW LEVEL SECURITY;

-- سیاست‌های RLS پایه (کاربران احراز هویت‌شده می‌توانند بخوانند)
CREATE POLICY "Authenticated read" ON users FOR SELECT TO authenticated USING (true);
CREATE POLICY "Authenticated read" ON customers FOR SELECT TO authenticated USING (true);
CREATE POLICY "Authenticated read" ON vessels FOR SELECT TO authenticated USING (true);
CREATE POLICY "Authenticated read" ON engines FOR SELECT TO authenticated USING (true);
CREATE POLICY "Authenticated read" ON work_orders FOR SELECT TO authenticated USING (true);
CREATE POLICY "Authenticated read" ON parts FOR SELECT TO authenticated USING (true);
CREATE POLICY "Authenticated read" ON invoices FOR SELECT TO authenticated USING (true);
CREATE POLICY "Authenticated read" ON payments FOR SELECT TO authenticated USING (true);
CREATE POLICY "Authenticated read" ON chat_conversations FOR SELECT TO authenticated USING (true);
CREATE POLICY "Authenticated read" ON chat_messages FOR SELECT TO authenticated USING (true);

-- اجازه نوشتن برای کاربران احراز هویت‌شده
CREATE POLICY "Authenticated insert" ON customers FOR INSERT TO authenticated WITH CHECK (true);
CREATE POLICY "Authenticated update" ON customers FOR UPDATE TO authenticated USING (true);
CREATE POLICY "Authenticated insert" ON vessels FOR INSERT TO authenticated WITH CHECK (true);
CREATE POLICY "Authenticated update" ON vessels FOR UPDATE TO authenticated USING (true);
CREATE POLICY "Authenticated insert" ON engines FOR INSERT TO authenticated WITH CHECK (true);
CREATE POLICY "Authenticated update" ON engines FOR UPDATE TO authenticated USING (true);
CREATE POLICY "Authenticated insert" ON work_orders FOR INSERT TO authenticated WITH CHECK (true);
CREATE POLICY "Authenticated update" ON work_orders FOR UPDATE TO authenticated USING (true);
CREATE POLICY "Authenticated insert" ON parts FOR INSERT TO authenticated WITH CHECK (true);
CREATE POLICY "Authenticated update" ON parts FOR UPDATE TO authenticated USING (true);
CREATE POLICY "Authenticated insert" ON invoices FOR INSERT TO authenticated WITH CHECK (true);
CREATE POLICY "Authenticated update" ON invoices FOR UPDATE TO authenticated USING (true);
CREATE POLICY "Authenticated insert" ON payments FOR INSERT TO authenticated WITH CHECK (true);
CREATE POLICY "Authenticated insert" ON chat_conversations FOR INSERT TO authenticated WITH CHECK (true);
CREATE POLICY "Authenticated insert" ON chat_messages FOR INSERT TO authenticated WITH CHECK (true);
```

### مرحله ۵: ساخت Supabase Client
فایل `src/lib/supabase.ts`:
```typescript
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)
```

### مرحله ۶: افزودن Authentication
از Supabase Auth برای ورود استفاده کنید:
```typescript
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'password'
})
```

---

## ۳. ساخت اکانت سوپر ادمین اول

پس از راه‌اندازی، اولین کاربر سوپر ادمین را در Supabase SQL Editor ایجاد کنید:

```sql
INSERT INTO users (name, email, role, status, password)
VALUES ('Admin', 'admin@marineforce.io', 'super_admin', 'active', 'your_password_hash');
```

---

## نکته مهم
در حال حاضر اپلیکیشن با **localStorage** کار می‌کند و نیازی به سرور ندارد.
برای استفاده با Supabase، باید store (Zustand) را به Supabase متصل کنید —
یعنی به‌جای `localStorage`، داده‌ها از جداول Supabase خوانده و نوشته شوند.
این کار نیازمند بازنویسی لایه داده است.
