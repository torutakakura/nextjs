# App Directory と tailwindcss で環境を作成する方法

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

```bash
$ npm run dev
```
