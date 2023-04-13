# NEXTjs を supabase に繋ぎこむ

## supabase

1. ログインしてプロジェクトを作成する。
   ※ データベースのパスワードは保存しておくこと。

2. SQL Editor 　＞　 New query でテーブル作成を行う

```sql
-- user_detailsテーブル作成
create table user_details (
  id uuid primary key references auth.users on delete cascade,
  name text,
  email text not null,
  phone text not null,
  address text not null,
  avatar_url text
);

-- user_detailsテーブルRLS設定
alter table user_details enable row level security;
create policy "プロフィールは誰でも参照可能" on user_details for select using ( true );
create policy "自身のプロフィールを更新" on user_details for update using (auth.uid() = id);

-- サインアップ時にプロフィールテーブル作成する関数
create function public.handle_new_user()
returns trigger as $$
begin
  insert into public.profiles (id, email)
  values (new.id, new.email);
  return new;
end;
$$ language plpgsql security definer set search_path = public;

-- サインアップ時にプロフィールテーブル作成する関数を呼び出すトリガー
create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();

```

## NEXTjs

### App Directory と tailwindcss でプロジェクト作成する

1. NEXTjs でプロジェクト作成

```bash
$ npx create-next-app <プロジェクト名>
```

以下の内容でプロジェクトでプロジェクトを作成する。

- Would you like to use TypeScript with this project? ... Yes
- Would you like to use ESLint with this project? ... Yes
- Would you like to use Tailwind CSS with this project? ... Yes
- Would you like to use `src/` directory with this project? ... No
- Would you like to use experimental `app/` directory with this project? ... Yes
- What import alias would you like configured? ... @/\*

2. api ディレクトリは不要なので削除

3. tailwindcss.config.js で不要な記載を削除

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./pages/**/*.{js,ts,jsx,tsx}",
    "./components/**/*.{js,ts,jsx,tsx}",
    "./app/**/*.{js,ts,jsx,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
};
```

4. globals.css で不要な記載を削除する

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

5. page.tsx を編集して、動作確認

```tsx
export default function Home() {
  return <div className="text-center">test</div>;
}
```

### supabase で作成した DB の型情報を取得

1. 必要なモジュールをインストール

```bash
$ npm i -D @supabase/auth-helpers-nextjs @supabase/supabase-js @types/uuid date-fns server-only sharp supabase uuid zustand encoding
```

※ もし、動かない場合はバージョンを合わせる

```json
{
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "@supabase/auth-helpers-nextjs": "^0.5.4",
    "@supabase/supabase-js": "^2.4.1",
    "@types/uuid": "^9.0.0",
    "date-fns": "^2.29.3",
    "next": "13.1.4",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "server-only": "^0.0.1",
    "sharp": "^0.31.3",
    "supabase": "^1.34.5",
    "uuid": "^9.0.0",
    "zustand": "^4.3.2"
  },
  "devDependencies": {
    "@types/node": "18.11.3",
    "@types/react": "18.0.21",
    "@types/react-dom": "18.0.6",
    "autoprefixer": "^10.4.12",
    "encoding": "^0.1.13",
    "postcss": "^8.4.18",
    "tailwindcss": "^3.2.4",
    "typescript": "4.9.4"
  }
}
```

2. .env ファイルを作成

```
NEXT_PUBLIC_SUPABASE_URL=YOUR_SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY=YOUR_SUPABASE_ANON_KEY
```

※ YOUR_SUPABASE_URL, YOUR_SUPABASE_ANON_KEY は supabase のプロジェクト > 設定 > API から確認

3. supabase に CLI でログイン

```bash
$ npx supabase login
```

- 表示された URL に遷移して token を生成する。
- token をコピーして CLI に貼り付けてログインを行う。

4. supabase プロジェクトの初期化

```bash
$ npx supabase init
```

5. supabase のプロジェクト > 設定 > General から ReferenceID をコピーしてデータベースに接続

```bash
$ npx supabase link --project-ref ＜コピーしたReferenceID＞
```

- データベースのパスワードを聞かれるので、保管している DB のパスワードを貼りつける

6. utils フォルダーを作成して、データベースの型情報を抽出.。

```bash
$ mkdir utils
$ npx supabase gen types --typescript --linked > utils/database.types.ts
```

### NEXTjs と supabase を繋ぎこむ

1. utils/supabase-server.ts を作成

```ts
// Supabase Client
import { headers, cookies } from "next/headers";
import { createServerComponentSupabaseClient } from "@supabase/auth-helpers-nextjs";
import type { Database } from "../utils/database.types";

export const createClient = () =>
  createServerComponentSupabaseClient<Database>({
    headers,
    cookies,
  });
```

2. utils/supabase-browser.ts を作成

```ts
// Supabase Client
import { createBrowserSupabaseClient } from "@supabase/auth-helpers-nextjs";
import { Database } from "./database.types";

export const createClient = () => createBrowserSupabaseClient<Database>();
```

3. app/components/supabase-listener.ts を作成

```ts
"use client";

import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { useSupabase } from "./supabase-provider";
import useStore from "../../store";

// ユーザーがログインまたはログアウトするたびに新しいセッションを取得する
const SupabaseListener = ({
  serverAccessToken,
}: {
  serverAccessToken?: string;
}) => {
  const router = useRouter();
  const { setUser } = useStore();
  const { supabase } = useSupabase();

  useEffect(() => {
    // セッション情報取得
    const getSession = async () => {
      const { data } = await supabase.auth.getSession();

      if (data.session) {
        // ユーザーIDにとメールアドレスを状態管理に設定
        setUser({
          id: data.session.user.id,
          email: data.session.user.email,
        });
      }
    };
    // リフレッシュ時にセッション情報取得
    getSession();

    // ログイン、ログアウトした時に認証状態を監視
    const {
      data: { subscription },
    } = supabase.auth.onAuthStateChange((event, session) => {
      setUser({ id: session?.user.id, email: session?.user.email });

      // アクセストークンチェック
      if (session?.access_token !== serverAccessToken) {
        // キャッシュクリア
        router.refresh();
      }
    });

    return () => {
      subscription.unsubscribe();
    };
  }, [serverAccessToken, router, supabase]);

  return null;
};

export default SupabaseListener;
```

4. app/components/supabase-provider.ts を作成

```ts
"use client";

import { createContext, useContext, useState } from "react";
import { createClient } from "../../utils/supabase-browser";

import type { SupabaseClient } from "@supabase/auth-helpers-nextjs";
import type { Database } from "../../utils/database.types";

type SupabaseContext = {
  supabase: SupabaseClient<Database>;
};

// コンテキスト
const Context = createContext<SupabaseContext>(null!);

// プロバイダー
export default function SupabaseProvider({
  children,
}: {
  children: React.ReactNode;
}) {
  const [supabase] = useState(() => createClient());

  return (
    <Context.Provider value={{ supabase }}>
      <>{children}</>
    </Context.Provider>
  );
}

// Supabaseクライアント
export const useSupabase = () => useContext(Context);
```

5. store/index.ts を作成

```ts
// Zustand Reactの状態管理ライブラリ
// https://github.com/pmndrs/zustand
import { create } from "zustand";

type User = {
  id: string | undefined;
  email: string | undefined;
};

type State = {
  user: User;
  setUser: (payload: User) => void;
};

const useStore = create<State>((set) => ({
  // 初期値
  user: { id: "", email: "" },
  // アップデート
  setUser: (payload) => set({ user: payload }),
}));

export default useStore;
```

6. app/layout.tsx を作成

```tsx
import "server-only";

import SupabaseListener from "./components/supabase-listener";
import SupabaseProvider from "./components/supabase-provider";
import "../styles/globals.css";
import Navigation from "./components/navigation";
import { createClient } from "../utils/supabase-server";

// キャッシュをしない
export const revalidate = 0;

// レイアウト
const RootLayout = async ({ children }: { children: React.ReactNode }) => {
  const supabase = createClient();

  // セッション情報取得
  const {
    data: { session },
  } = await supabase.auth.getSession();

  return (
    <html>
      <body>
        <SupabaseProvider>
          <SupabaseListener serverAccessToken={session?.access_token} />
          <main className="flex-1 container max-w-screen-xl mx-auto px-5 py-10">
            {children}
          </main>
        </SupabaseProvider>
      </body>
    </html>
  );
};

export default RootLayout;
```

7. app/page.tsx で編集して、動作確認

```tsx
import { createClient } from "@/utils/supabase-server";
import { notFound } from "next/navigation";

export default async function Home() {
  const supabase = createClient();

  const { data: userList } = await supabase
    .from("user_details")
    .select()
    .order("created_at", { ascending: false });

  if (!userList) return notFound();

  return (
    <>
      {userList.map((user, idx) => (
        <div key={idx}>{user.name}</div>
      ))}
    </>
  );
}
```

```bash
$ npm run dev
```
