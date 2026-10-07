# راهنمای کامل اتصال MarineForce به Supabase

## مرحله ۱: ساخت پروژه Supabase

1. به [supabase.com](https://supabase.com) بروید و ثبت‌نام کنید
2. روی **New Project** کلیک کنید
3. نام پروژه: `marineforce`
4. رمز دیتابیس را یادداشت کنید
5. منطقه (Region) را انتخاب کنید
6. روی **Create** کلیک کنید و منتظر بمانید (~۲ دقیقه)

## مرحله ۲: دریافت کلیدها

1. در پنل Supabase به **Settings → API** بروید
2. این دو مقدار را کپی کنید:
   - **Project URL**: `https://xxxxx.supabase.co`
   - **anon public key**: `eyJhbGciOi...` (طولانی)

## مرحله ۳: ساخت جداول دیتابیس

1. در پنل Supabase به **SQL Editor** بروید
2. اسکریپت زیر را کپی و اجرا کنید (Run):

```sql
-- ============================================
-- MarineForce Database Schema
-- ============================================

-- کاربران
CREATE TABLE IF NOT EXISTS users (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  role TEXT NOT NULL DEFAULT 'read_only',
  status TEXT DEFAULT 'active',
  password TEXT,
  avatar_color TEXT DEFAULT '#06b6d4',
  supplier_id UUID,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- مشتریان
CREATE TABLE IF NOT EXISTS customers (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  customer_id TEXT UNIQUE,
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  company TEXT,
  email TEXT,
  phone TEXT,
  alt_phone TEXT,
  address TEXT,
  country TEXT DEFAULT 'ایران',
  city TEXT,
  type TEXT DEFAULT 'Individual',
  tax_id TEXT,
  notes TEXT,
  status TEXT DEFAULT 'active',
  password TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- شناورها
CREATE TABLE IF NOT EXISTS vessels (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  vessel_id TEXT UNIQUE,
  name TEXT NOT NULL,
  registration_number TEXT,
  hin TEXT,
  type TEXT,
  manufacturer TEXT,
  model TEXT,
  year INTEGER,
  length INTEGER,
  hull_material TEXT,
  location TEXT,
  owner_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  notes TEXT,
  status TEXT DEFAULT 'active',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- موتورها
CREATE TABLE IF NOT EXISTS engines (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  engine_id TEXT UNIQUE,
  manufacturer TEXT,
  model TEXT,
  serial_number TEXT,
  type TEXT,
  horsepower INTEGER,
  fuel_type TEXT,
  year INTEGER,
  hours INTEGER,
  installation_date TEXT,
  location TEXT,
  vessel_id UUID REFERENCES vessels(id) ON DELETE SET NULL,
  warranty_status TEXT DEFAULT 'none',
  ecu_info TEXT,
  notes TEXT,
  status TEXT DEFAULT 'active',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- تاریخچه سرویس موتور
CREATE TABLE IF NOT EXISTS engine_service_history (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  engine_id UUID REFERENCES engines(id) ON DELETE CASCADE,
  date TEXT,
  type TEXT,
  description TEXT,
  technician TEXT,
  hours INTEGER,
  cost INTEGER DEFAULT 0,
  notes TEXT
);

-- درخواست‌های سرویس
CREATE TABLE IF NOT EXISTS service_requests (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  request_number TEXT UNIQUE,
  customer_id UUID REFERENCES customers(id),
  vessel_id UUID REFERENCES vessels(id),
  engine_id UUID REFERENCES engines(id),
  type TEXT,
  description TEXT,
  priority TEXT DEFAULT 'Normal',
  status TEXT DEFAULT 'New',
  assigned_technician_id UUID,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expected_completion TEXT
);

-- تکنسین‌ها
CREATE TABLE IF NOT EXISTS technicians (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT,
  phone TEXT,
  specialty TEXT,
  experience INTEGER DEFAULT 0,
  hourly_rate INTEGER DEFAULT 0,
  status TEXT DEFAULT 'available',
  skills JSONB DEFAULT '[]',
  certifications JSONB DEFAULT '[]',
  completed_jobs INTEGER DEFAULT 0,
  rating REAL DEFAULT 5,
  avatar_color TEXT DEFAULT '#06b6d4',
  password TEXT
);

-- سفارشات کار
CREATE TABLE IF NOT EXISTS work_orders (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  work_order_number TEXT UNIQUE,
  customer_id UUID REFERENCES customers(id),
  vessel_id UUID REFERENCES vessels(id),
  engine_id UUID REFERENCES engines(id),
  complaint TEXT,
  inspection TEXT,
  diagnosis TEXT,
  work_performed TEXT,
  assigned_technician_id UUID,
  items JSONB DEFAULT '[]',
  labor JSONB DEFAULT '[]',
  additional_charges INTEGER DEFAULT 0,
  discount INTEGER DEFAULT 0,
  tax_rate INTEGER DEFAULT 9,
  estimated_cost INTEGER,
  final_cost INTEGER,
  warranty JSONB,
  internal_notes TEXT,
  customer_notes TEXT,
  status TEXT DEFAULT 'Open',
  status_history JSONB DEFAULT '[]',
  scheduled_date TEXT,
  scheduled_time TEXT,
  scheduled_duration INTEGER,
  scheduled_status TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- انبارها
CREATE TABLE IF NOT EXISTS warehouses (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  location TEXT,
  manager TEXT
);

-- قطعات
CREATE TABLE IF NOT EXISTS parts (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  sku TEXT UNIQUE,
  part_number TEXT,
  name TEXT NOT NULL,
  manufacturer TEXT,
  category TEXT,
  description TEXT,
  supplier_id UUID,
  cost_price INTEGER DEFAULT 0,
  selling_price INTEGER DEFAULT 0,
  quantity INTEGER DEFAULT 0,
  min_quantity INTEGER DEFAULT 1,
  max_quantity INTEGER DEFAULT 10,
  warehouse_id UUID,
  location TEXT,
  barcode TEXT,
  warranty_months INTEGER,
  owner_type TEXT DEFAULT 'workshop',
  status TEXT DEFAULT 'active'
);

-- تأمین‌کنندگان
CREATE TABLE IF NOT EXISTS suppliers (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  company TEXT NOT NULL,
  contact_person TEXT,
  phone TEXT,
  email TEXT,
  address TEXT,
  tax_id TEXT,
  payment_terms TEXT DEFAULT 'Net 30',
  notes TEXT,
  outstanding_amount INTEGER DEFAULT 0,
  total_purchases INTEGER DEFAULT 0,
  total_paid INTEGER DEFAULT 0,
  balance_type TEXT DEFAULT 'settled',
  status TEXT DEFAULT 'active',
  password TEXT
);

-- فاکتورها
CREATE TABLE IF NOT EXISTS invoices (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  invoice_number TEXT UNIQUE,
  customer_id UUID REFERENCES customers(id),
  work_order_id UUID,
  issue_date TEXT,
  due_date TEXT,
  items JSONB DEFAULT '[]',
  labor_total INTEGER DEFAULT 0,
  parts_total INTEGER DEFAULT 0,
  tax_rate INTEGER DEFAULT 9,
  discount INTEGER DEFAULT 0,
  total INTEGER DEFAULT 0,
  paid INTEGER DEFAULT 0,
  remaining INTEGER DEFAULT 0,
  status TEXT DEFAULT 'Draft',
  notes TEXT
);

-- پرداخت‌ها
CREATE TABLE IF NOT EXISTS payments (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  payment_number TEXT UNIQUE,
  invoice_id UUID REFERENCES invoices(id),
  customer_id UUID,
  amount INTEGER,
  method TEXT,
  reference TEXT,
  date TEXT,
  received_by TEXT,
  notes TEXT
);

-- هزینه‌ها
CREATE TABLE IF NOT EXISTS expenses (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  category TEXT,
  amount INTEGER,
  date TEXT,
  supplier TEXT,
  description TEXT,
  created_by TEXT
);

-- قرارها
CREATE TABLE IF NOT EXISTS appointments (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  customer_id UUID,
  vessel_id UUID,
  service TEXT,
  technician_id UUID,
  date TEXT,
  time TEXT,
  duration INTEGER DEFAULT 1,
  status TEXT DEFAULT 'scheduled',
  notes TEXT
);

-- لاگ ممیزی
CREATE TABLE IF NOT EXISTS audit_logs (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  user_name TEXT,
  action TEXT,
  module TEXT,
  record_id TEXT,
  description TEXT,
  severity TEXT DEFAULT 'info'
);

-- ثبت‌نام‌ها
CREATE TABLE IF NOT EXISTS registrations (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT NOT NULL,
  password TEXT,
  phone TEXT,
  requested_role TEXT,
  status TEXT DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- تنظیمات سازمان
CREATE TABLE IF NOT EXISTS organization_settings (
  id INTEGER DEFAULT 1 PRIMARY KEY,
  company_name TEXT DEFAULT 'مرین‌فورس',
  address TEXT,
  phone TEXT,
  email TEXT,
  website TEXT,
  tax_id TEXT,
  currency TEXT DEFAULT 'IRR',
  timezone TEXT DEFAULT 'Asia/Tehran',
  invoice_prefix TEXT DEFAULT 'INV-2026-',
  work_order_prefix TEXT DEFAULT 'WO-2026-',
  customer_id_prefix TEXT DEFAULT 'CUST-',
  next_invoice_number INTEGER DEFAULT 1,
  next_work_order_number INTEGER DEFAULT 1,
  next_customer_number INTEGER DEFAULT 1,
  tax_rate INTEGER DEFAULT 9
);

-- چت
CREATE TABLE IF NOT EXISTS chat_conversations (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  name TEXT NOT NULL,
  participant_ids JSONB DEFAULT '[]',
  created_by UUID,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_message_at TIMESTAMPTZ DEFAULT NOW(),
  last_message_text TEXT,
  status TEXT DEFAULT 'active'
);

CREATE TABLE IF NOT EXISTS chat_messages (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  conversation_id UUID REFERENCES chat_conversations(id) ON DELETE CASCADE,
  sender_id UUID,
  sender_name TEXT,
  text TEXT,
  timestamp TIMESTAMPTZ DEFAULT NOW(),
  deleted_by_user BOOLEAN DEFAULT FALSE,
  edited_by_admin BOOLEAN DEFAULT FALSE,
  original_text TEXT
);

-- محدودیت چت
CREATE TABLE IF NOT EXISTS chat_restrictions (
  user_id UUID PRIMARY KEY,
  allowed_ids JSONB DEFAULT '[]'
);

-- ============================================
-- Row Level Security (RLS)
-- ============================================

-- فعال‌سازی RLS برای همه جداول
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE vessels ENABLE ROW LEVEL SECURITY;
ALTER TABLE engines ENABLE ROW LEVEL SECURITY;
ALTER TABLE engine_service_history ENABLE ROW LEVEL SECURITY;
ALTER TABLE service_requests ENABLE ROW LEVEL SECURITY;
ALTER TABLE technicians ENABLE ROW LEVEL SECURITY;
ALTER TABLE work_orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE warehouses ENABLE ROW LEVEL SECURITY;
ALTER TABLE parts ENABLE ROW LEVEL SECURITY;
ALTER TABLE suppliers ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
ALTER TABLE expenses ENABLE ROW LEVEL SECURITY;
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE registrations ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_conversations ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_messages ENABLE ROW LEVEL SECURITY;

-- سیاست‌ها: کاربران احراز هویت‌شده می‌توانند بخوانند و بنویسند
-- (در تولید، این سیاست‌ها را محدودتر کنید)

CREATE POLICY "read_all" ON users FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all_users" ON users FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON customers FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON customers FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON vessels FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON vessels FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON engines FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON engines FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON engine_service_history FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON engine_service_history FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON service_requests FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON service_requests FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON technicians FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON technicians FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON work_orders FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON work_orders FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON warehouses FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON warehouses FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON parts FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON parts FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON suppliers FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON suppliers FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON invoices FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON invoices FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON payments FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON payments FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON expenses FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON expenses FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON appointments FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON appointments FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON audit_logs FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON audit_logs FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON registrations FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON registrations FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON chat_conversations FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON chat_conversations FOR ALL TO authenticated USING (true) WITH CHECK (true);

CREATE POLICY "read_all" ON chat_messages FOR SELECT TO authenticated USING (true);
CREATE POLICY "write_all" ON chat_messages FOR ALL TO authenticated USING (true) WITH CHECK (true);

-- ============================================
-- داده اولیه
-- ============================================

-- تنظیمات پیش‌فرض
INSERT INTO organization_settings (id, company_name, currency, timezone)
VALUES (1, 'مرین‌فورس', 'IRR', 'Asia/Tehran')
ON CONFLICT (id) DO NOTHING;

-- انبار پیش‌فرض
INSERT INTO warehouses (id, name, location)
VALUES (
  '00000000-0000-0000-0000-000000000001',
  'انبار اصلی',
  'کارگاه'
)
ON CONFLICT DO NOTHING;
```

## مرحله ۴: ساخت اکانت سوپر ادمین اول

در SQL Editor این را اجرا کنید (رمز را تغییر دهید):

```sql
INSERT INTO users (name, email, role, status, password, avatar_color)
VALUES (
  'مدیر سیستم',
  'admin@marineforce.io',
  'super_admin',
  'active',
  'admin1234',
  '#06b6d4'
);
```

## مرحله ۵: ساخت فایل `.env.local`

در پوشه اصلی پروژه فایل `.env.local` بسازید:

```
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOi...
```

مقادیر را با کلیدهایی که در مرحله ۲ کپی کردید جایگزین کنید.

## مرحله ۶: نصب Supabase Client

```bash
npm install @supabase/supabase-js
```

## مرحله ۷: ساخت فایل Supabase Client

فایل `src/lib/supabase.ts` بسازید:

```typescript
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL!
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

export const supabase = createClient(supabaseUrl, supabaseAnonKey)
```

## مرحله ۸: اجرای پروژه

```bash
npm install
npm run dev
```

سپس به `http://localhost:3000` بروید.

---

## نکات مهم

1. **احراز هویت:** در حال حاضر اپلیکیشن از localStorage استفاده می‌کند. برای اتصال به Supabase Auth، باید `login()` و `registerUser()` در store را با `supabase.auth.signInWithPassword()` جایگزین کنید.

2. **خواندن/نوشتن داده:** باید توابع store (مثل `addCustomer`, `updateCustomer`) را به‌جای استفاده از `set()` Zustand، به `supabase.from('customers').insert()` تغییر دهید.

3. **Realtime:** برای چت زنده واقعی، از `supabase.channel().on('postgres_changes')` استفاده کنید.

4. **امنیت:** سیاست‌های RLS در مرحله ۳ برای شروع مناسب هستند. در تولید، باید محدودتر شوند (مثلاً کاربر فقط داده‌های خودش را ببیند).

5. **ذخیره فایل:** هر فایلی که در Supabase Storage آپلود می‌شود نیاز به ساخت Bucket دارد.
