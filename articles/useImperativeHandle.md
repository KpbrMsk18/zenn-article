---
title: "ReactのuseImperativeHandleを理解する：子コンポーネントの機能を「公開」するとは？"
emoji: "⚛️"
type: "tech"
topics: ["react", "hooks", "typescript", "frontend"]
published: false
---

# React の useImperativeHandle とは？

`useImperativeHandle`は、React の関数コンポーネントにおいて、親コンポーネントから子コンポーネントの内部機能にアクセスできるようにする Hook です。

通常、React では**データは上から下へ（親から子へ）**流れるという原則がありますが、`useImperativeHandle`を使うことで、親コンポーネントが子コンポーネントの特定の機能を呼び出すことができるようになります。

## 基本的な構文

```typescript
useImperativeHandle(ref, createHandle, [deps]);
```

- `ref`: 親から渡される`ref`オブジェクト
- `createHandle`: 公開したい機能を返す関数
- `deps`: 依存配列（省略可能）

## 「公開する」とはどういうこと？

React の関数コンポーネントは通常、内部の状態や関数を外部から直接操作することはできません。しかし、`useImperativeHandle`を使うことで、コンポーネントの**特定の機能だけを選択的に外部に公開**できます。

## useImperativeHandle で公開すると何が嬉しいの？

### 1. **コンポーネントの再利用性が向上する**

通常の props では実現が困難な、より柔軟なコンポーネント操作が可能になります。

```typescript
// 悪い例：propsで全ての操作を制御しようとする場合
<CustomInput
  shouldFocus={shouldFocus}
  shouldClear={shouldClear}
  onValueRequest={handleValueRequest}
/>

// 良い例：useImperativeHandleを使用
<CustomInput ref={inputRef} />
// 必要な時に直接操作
inputRef.current?.focus();
```

### 2. **複雑な状態管理を避けられる**

親コンポーネントで複雑な状態を管理する必要がなくなり、コードがシンプルになります。

```typescript
// 従来の方法では親で状態管理が必要
const [shouldShowModal, setShouldShowModal] = useState(false);
const [modalContent, setModalContent] = useState("");

// useImperativeHandleを使えば直接操作
modalRef.current?.show("エラーが発生しました");
modalRef.current?.hide();
```

### 3. **命令的な操作が自然に表現できる**

フォーカス、スクロール、アニメーションなど、本質的に命令的な操作を自然に表現できます。

```typescript
// フォーカス制御の例
const handleSubmit = () => {
  if (!isValid) {
    errorInputRef.current?.focus(); // エラーがある入力欄にフォーカス
    errorInputRef.current?.highlight(); // 入力欄をハイライト
  }
};
```

### 4. **第三者ライブラリとの統合が簡単**

外部ライブラリの API を直接呼び出すような操作を、React コンポーネントから簡単に実行できます。

```typescript
// 地図ライブラリの例
const MapComponent = forwardRef<MapHandle, MapProps>((props, ref) => {
  const mapInstance = useRef<MapAPI>(null);

  useImperativeHandle(ref, () => ({
    zoomTo: (level: number) => mapInstance.current?.setZoom(level),
    panTo: (lat: number, lng: number) => mapInstance.current?.panTo(lat, lng),
    addMarker: (marker: Marker) => mapInstance.current?.addMarker(marker),
  }));

  // 使用側
  mapRef.current?.zoomTo(15);
  mapRef.current?.panTo(35.6762, 139.6503);
});
```

### 5. **パフォーマンスの向上**

不要な再レンダリングを避けることができます。

```typescript
// propsで制御する場合、親の状態変更で子が再レンダリング
const [triggerAction, setTriggerAction] = useState(0);

// useImperativeHandleなら再レンダリングなしで操作可能
const handleAction = () => {
  childRef.current?.performAction(); // 再レンダリングなし
};
```

### 例：入力フィールドの公開

```typescript
import React, { useImperativeHandle, useRef, forwardRef } from "react";

// 公開したい機能の型定義
export interface InputHandle {
  focus: () => void;
  clear: () => void;
  getValue: () => string;
}

const CustomInput = forwardRef<InputHandle, {}>((props, ref) => {
  const inputRef = useRef<HTMLInputElement>(null);

  // 親コンポーネントに公開する機能を定義
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current?.focus();
    },
    clear: () => {
      if (inputRef.current) {
        inputRef.current.value = "";
      }
    },
    getValue: () => {
      return inputRef.current?.value || "";
    },
  }));

  return <input ref={inputRef} type="text" />;
});

export default CustomInput;
```

### 親コンポーネントでの使用

```typescript
import React, { useRef } from "react";
import CustomInput, { InputHandle } from "./CustomInput";

const ParentComponent = () => {
  const inputRef = useRef<InputHandle>(null);

  const handleFocus = () => {
    inputRef.current?.focus();
  };

  const handleClear = () => {
    inputRef.current?.clear();
  };

  const handleGetValue = () => {
    const value = inputRef.current?.getValue();
    console.log("現在の値:", value);
  };

  return (
    <div>
      <CustomInput ref={inputRef} />
      <button onClick={handleFocus}>フォーカス</button>
      <button onClick={handleClear}>クリア</button>
      <button onClick={handleGetValue}>値を取得</button>
    </div>
  );
};
```

## 主な使用用途

### 1. フォーム操作

- 入力フィールドのフォーカス制御
- バリデーション結果の取得
- フォームデータのクリア

### 2. メディア制御

- 動画や音声の再生・停止
- シークバーの制御

### 3. アニメーション制御

- アニメーションの開始・停止
- 特定のタイミングでのアニメーション実行

### 4. 複雑なコンポーネントの操作

- モーダルの開閉
- ドロワーメニューの制御
- カスタムコンポーネントの内部状態操作

## 実践例：動画プレーヤーコンポーネント

```typescript
import React, { useImperativeHandle, useRef, forwardRef } from "react";

export interface VideoPlayerHandle {
  play: () => void;
  pause: () => void;
  setCurrentTime: (time: number) => void;
  getCurrentTime: () => number;
  getDuration: () => number;
}

interface VideoPlayerProps {
  src: string;
}

const VideoPlayer = forwardRef<VideoPlayerHandle, VideoPlayerProps>(
  ({ src }, ref) => {
    const videoRef = useRef<HTMLVideoElement>(null);

    useImperativeHandle(ref, () => ({
      play: () => {
        videoRef.current?.play();
      },
      pause: () => {
        videoRef.current?.pause();
      },
      setCurrentTime: (time: number) => {
        if (videoRef.current) {
          videoRef.current.currentTime = time;
        }
      },
      getCurrentTime: () => {
        return videoRef.current?.currentTime || 0;
      },
      getDuration: () => {
        return videoRef.current?.duration || 0;
      },
    }));

    return (
      <video
        ref={videoRef}
        src={src}
        controls={false}
        style={{ width: "100%", height: "auto" }}
      />
    );
  }
);

export default VideoPlayer;
```

## 使用時の注意点

### 1. 過度な使用は避ける

`useImperativeHandle`は便利ですが、React の宣言的なパラダイムに反する場合があります。通常の props と state 管理で解決できる場合は、そちらを優先しましょう。

### 2. TypeScript での型安全性

公開する機能の型を明確に定義することで、型安全性を保つことができます。

### 3. React バージョンによる ref の扱いの違い

#### React 18 以前: forwardRef が必須

関数コンポーネントで ref を受け取るには、`forwardRef`が必要です。

```typescript
// React 18での書き方
const CustomInput = forwardRef<InputHandle, {}>((props, ref) => {
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
  }));

  return <input ref={inputRef} type="text" />;
});
```

#### React 19 以降: ref as prop

React 19 では、`ref`が通常の props として扱えるようになったため、`forwardRef`が不要になりました。

```typescript
// React 19での書き方
interface CustomInputProps {
  ref?: React.Ref<InputHandle>;
}

const CustomInput = ({ ref }: CustomInputProps) => {
  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
  }));

  return <input ref={inputRef} type="text" />;
};
```

詳細については[React 19 のリリースノート](https://ja.react.dev/blog/2024/12/05/react-19#ref-as-a-prop)を参照してください。

**注意**: この記事のコード例は React 18 ベースで書かれています。React 19 を使用する場合は、`forwardRef`の使用は任意になります。

## まとめ

`useImperativeHandle`は、子コンポーネントの特定の機能を親コンポーネントから操作したい場合に有用な Hook です。

- **「公開する」**とは、コンポーネント内部の機能を外部から呼び出せるようにすること
- フォーム操作、メディア制御、アニメーション制御などで活用
- 過度な使用は避け、適切な場面で使用することが重要
- TypeScript と組み合わせることで型安全な開発が可能

適切に使用することで、より柔軟で再利用可能なコンポーネントを作成できます。
