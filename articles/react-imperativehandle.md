---
title: "【React 19】useImperativeHandleが格段に使いやすくなった！子コンポーネントの機能を「公開」する方法"
emoji: "⚛️"
type: "tech"
topics: ["react", "hooks", "typescript", "frontend", "react19"]
published: false
---

# React 19 で useImperativeHandle が劇的に簡単になった！

React 19 で`useImperativeHandle`の使い方が大幅に簡素化されました。これまで React 18 では`forwardRef`という複雑な概念を理解する必要がありましたが、**React 19 では通常の props と同じように`ref`を扱えるようになり、格段に使いやすくなりました**。

`useImperativeHandle`は、親コンポーネントから子コンポーネントの内部機能にアクセスできるようにする Hook です。通常の React の「上から下へのデータフロー」とは逆方向の操作を可能にします。

## 基本的な構文

```typescript
useImperativeHandle(ref, createHandle, [deps]);
```

- `ref`: 親から渡される`ref`オブジェクト
- `createHandle`: 公開したい機能を返す関数
- `deps`: 依存配列（省略可能）

## React 19 での劇的な変化：ref as prop

### React 18 以前の複雑さ

React 18 では、関数コンポーネントで ref を受け取るために`forwardRef`という特別な関数でラップする必要がありました：

```typescript
// React 18での複雑な書き方
import { forwardRef, useImperativeHandle, useRef } from "react";

const CustomInput = forwardRef<InputHandle, {}>((props, ref) => {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
  }));

  return <input ref={inputRef} type="text" />;
});
```

### React 19 での簡潔な書き方

React 19 では、`ref`が通常の props として扱えるため、`forwardRef`が不要になりました：

```typescript
// React 19でのシンプルな書き方
import { useImperativeHandle, useRef } from "react";

interface CustomInputProps {
  ref?: React.Ref<InputHandle>;
}

const CustomInput = ({ ref }: CustomInputProps) => {
  const inputRef = useRef<HTMLInputElement>(null);

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current?.focus(),
  }));

  return <input ref={inputRef} type="text" />;
};
```

**これだけでも、学習コストが大幅に下がりました！**

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
interface MapComponentProps {
  ref?: React.Ref<MapHandle>;
}

const MapComponent = ({ ref }: MapComponentProps) => {
  const mapInstance = useRef<MapAPI>(null);

  useImperativeHandle(ref, () => ({
    zoomTo: (level: number) => mapInstance.current?.setZoom(level),
    panTo: (lat: number, lng: number) => mapInstance.current?.panTo(lat, lng),
    addMarker: (marker: Marker) => mapInstance.current?.addMarker(marker),
  }));

  // 使用側
  // mapRef.current?.zoomTo(15);
  // mapRef.current?.panTo(35.6762, 139.6503);
};
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

## 実践例：入力フィールドコンポーネント

### コンポーネント定義

```typescript
import { useImperativeHandle, useRef } from "react";

// 公開したい機能の型定義
export interface InputHandle {
  focus: () => void;
  clear: () => void;
  getValue: () => string;
  setValue: (value: string) => void;
}

interface CustomInputProps {
  ref?: React.Ref<InputHandle>;
  placeholder?: string;
}

const CustomInput = ({ ref, placeholder }: CustomInputProps) => {
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
    setValue: (value: string) => {
      if (inputRef.current) {
        inputRef.current.value = value;
      }
    },
  }));

  return (
    <input
      ref={inputRef}
      type="text"
      placeholder={placeholder}
      className="border p-2 rounded"
    />
  );
};

export default CustomInput;
```

### 親コンポーネントでの使用

```typescript
import { useRef } from "react";
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

  const handleSetValue = () => {
    inputRef.current?.setValue("プリセット値");
  };

  return (
    <div className="space-y-4">
      <CustomInput ref={inputRef} placeholder="何か入力してください" />
      <div className="space-x-2">
        <button
          onClick={handleFocus}
          className="bg-blue-500 text-white px-4 py-2 rounded"
        >
          フォーカス
        </button>
        <button
          onClick={handleClear}
          className="bg-red-500 text-white px-4 py-2 rounded"
        >
          クリア
        </button>
        <button
          onClick={handleGetValue}
          className="bg-green-500 text-white px-4 py-2 rounded"
        >
          値を取得
        </button>
        <button
          onClick={handleSetValue}
          className="bg-purple-500 text-white px-4 py-2 rounded"
        >
          値をセット
        </button>
      </div>
    </div>
  );
};
```

## 実践例：動画プレーヤーコンポーネント

```typescript
import { useImperativeHandle, useRef } from "react";

export interface VideoPlayerHandle {
  play: () => void;
  pause: () => void;
  setCurrentTime: (time: number) => void;
  getCurrentTime: () => number;
  getDuration: () => number;
  setVolume: (volume: number) => void;
}

interface VideoPlayerProps {
  ref?: React.Ref<VideoPlayerHandle>;
  src: string;
}

const VideoPlayer = ({ ref, src }: VideoPlayerProps) => {
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
    setVolume: (volume: number) => {
      if (videoRef.current) {
        videoRef.current.volume = Math.max(0, Math.min(1, volume));
      }
    },
  }));

  return (
    <video
      ref={videoRef}
      src={src}
      controls={false}
      className="w-full h-auto"
    />
  );
};

export default VideoPlayer;
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

## 使用時の注意点

### 1. 過度な使用は避ける

`useImperativeHandle`は便利ですが、React の宣言的なパラダイムに反する場合があります。通常の props と state 管理で解決できる場合は、そちらを優先しましょう。

### 2. TypeScript での型安全性

公開する機能の型を明確に定義することで、型安全性を保つことができます。

### 3. React 18 からの移行

React 18 から移行する場合は、`forwardRef`を削除して、ref を props として受け取るように変更するだけです。

```typescript
// React 18 → React 19への移行例
// Before (React 18)
const Component = forwardRef<Handle, Props>((props, ref) => { ... });

// After (React 19)
interface ComponentProps extends Props {
  ref?: React.Ref<Handle>;
}
const Component = ({ ref, ...props }: ComponentProps) => { ... };
```

## なぜ今まで浸透しなかったのか？

1. **forwardRef の複雑さ**: React 18 以前では`forwardRef`の理解が必要で、学習コストが高かった
2. **React の哲学との矛盾**: 宣言的 UI の原則に反する命令的アプローチ
3. **ドキュメントでの扱い**: 「escape hatch」として紹介され、積極的に推奨されていなかった

**React 19 の`ref as prop`により、これらの障壁が大幅に低くなりました！**

## まとめ

React 19 で`useImperativeHandle`は格段に使いやすくなりました：

- **React 19**: `forwardRef`不要でシンプルな実装が可能
- **React 18**: `forwardRef`が必須で複雑だった
- **「公開する」**: コンポーネント内部の機能を外部から呼び出せるようにすること
- **活用場面**: フォーム操作、メディア制御、アニメーション制御など
- **注意点**: 過度な使用は避け、適切な場面で使用することが重要

React 19 の新機能により、これまで敬遠されがちだった`useImperativeHandle`が、より身近で実用的なツールになりました。適切に活用して、より柔軟で再利用可能なコンポーネントを作成しましょう！
