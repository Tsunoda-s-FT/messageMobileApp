# MessageMobileApp

### モバイルアプリケーションの概要

このモバイルアプリケーションは、学校から保護者へのコミュニケーションを目的としたメッセージングアプリです。アプリの主要な機能は、学校からのメッセージの受信、ブックマーク機能、リンク集の提供、および設定の管理です。React Nativeを使用して開発され、シンプルで直感的なユーザーインターフェースが特徴です。アプリはオフライン機能もサポートし、ユーザーがインターネット接続がない場合でもメッセージの閲覧が可能です。

### アーキテクチャ設計

#### プレゼンテーション層（UI層）

- **画面（Screens）**:
  - `HomeScreen`: メッセージの一覧表示。
  - `BookmarkScreen`: ブックマークされたメッセージの表示。
  - `LinkScreen`: 学校からのリンク集。
  - `SettingScreen`: アプリ設定とユーザープロファイルの編集。

- **UIコンポーネント（Components）**:
  - `MessageItem`: 個々のメッセージ表示。
  - `Header`: 画面上部のヘッダー。
  - `TabBar`: 画面下部のナビゲーションバー。

#### ロジック層（アプリケーション層）

- **コンテキスト（Contexts）**:
  - `MessageContext`: メッセージのデータと操作を管理。
  - `AuthContext`: ユーザー認証情報の管理。

- **サービス（Services）**:
  - `ApiService`: サーバーとの通信。
  - `StorageService`: ローカルストレージの操作。

- **ユーティリティ（Utilities）**:
  - データ変換やバリデーション関数。

#### データ層

- **ローカルデータベース**:
  - `AsyncStorage`: データのローカル保存。

- **API通信**:
  - サーバーとのデータ交換。

### ディレクトリ設計

```
/my-app
|-- /android
|-- /ios
|-- /src
    |-- /components
    |   |-- Header.js
    |   |-- MessageItem.js
    |   |-- BookmarkItem.js
    |   |-- LinkItem.js
    |   |-- ...
    |-- /screens
    |   |-- HomeScreen.js
    |   |-- BookmarkScreen.js
    |   |-- LinkScreen.js
    |   |-- SettingScreen.js
    |   |-- ...
    |-- /navigation
    |   |-- AppNavigator.js
    |   |-- ...
    |-- /context
    |   |-- MessageContext.js
    |   |-- AuthContext.js
    |   |-- ...
    |-- /services
    |   |-- ApiService.js
    |   |-- StorageService.js
    |   |-- ...
    |-- /utils
    |   |-- Constants.js
    |   |-- UtilityFunctions.js
    |   |-- ...
    |-- App.js
|-- /assets
    |-- /images
    |-- /fonts
    |-- ...
|-- app.json
|-- package.json
|-- ...
```

このディレクトリ設計は、アプリの構成要素を整理し、機能ごとに分類することで開発と保守を容易にします。また、アプリのスケールが拡大しても、新たな機能やコンポーネントを追加する際の基盤がしっかりしているため、将来の拡張にも対応しやすくなります。

以下、ロジック層とデータ層について考えます。

### MessageContext.js の詳細設計

`MessageContext.js`は、アプリケーションのメッセージ関連データとビジネスロジックを管理するための設計です。このコンテキストは、メッセージの取得、既読/未読の状態の更新、ブックマーク機能の管理などを提供します。

#### コメント付きコード

```javascript
import React, { createContext, useState, useEffect } from 'react';
import { ApiService } from '../services/ApiService';
import { StorageService } from '../services/StorageService';

// メッセージコンテキストの作成
export const MessageContext = createContext();

// メッセージプロバイダコンポーネント
export const MessageProvider = ({ children }) => {
  const [messages, setMessages] = useState([]); // メッセージのリスト
  const [bookmarks, setBookmarks] = useState([]); // ブックマークされたメッセージのリスト
  const [loading, setLoading] = useState(false); // ローディング状態
  const [error, setError] = useState(null); // エラー状態

  // メッセージをロードする関数
  const loadMessages = async () => {
    setLoading(true);
    try {
      let loadedMessages = await StorageService.getMessages();
      if (!loadedMessages || loadedMessages.length === 0) {
        // ローカルストレージにデータがない場合はAPIから取得
        loadedMessages = await ApiService.fetchMessages();
        await StorageService.saveMessages(loadedMessages);
      }
      setMessages(loadedMessages);
    } catch (err) {
      setError(err);
    } finally {
      setLoading(false);
    }
  };

  // メッセージの既読ステータスを更新する関数
  const markMessageAsRead = async (messageId) => {
    try {
      await ApiService.updateMessageStatus(messageId, true);
      setMessages(prevMessages =>
        prevMessages.map(message =>
          message.id === messageId ? { ...message, isRead: true } : message
        )
      );
    } catch (err) {
      setError(err);
    }
  };

  // メッセージをブックマークする関数
  const addBookmark = (messageId) => {
    const message = messages.find(m => m.id === messageId);
    if (message && !bookmarks.some(b => b.id === messageId)) {
      setBookmarks([...bookmarks, message]);
    }
  };

  // ブックマークを削除する関数
  const removeBookmark = (messageId) => {
    setBookmarks(bookmarks.filter(b => b.id !== messageId));
  };

  // コンポーネントのマウント時にメッセージをロード
  useEffect(() => {
    loadMessages();
  }, []);

  return (
    <MessageContext.Provider value={{
      messages,
      bookmarks,
      loading,
      error,
      markMessageAsRead,
      addBookmark,
      removeBookmark
    }}>
      {children}
    </MessageContext.Provider>
  );
};
```

### 解説

#### メッセージのロード

- `loadMessages`関数は、まずローカルストレージからメッセージを取得しようとします。もしローカルストレージにメッセージがない場合は、APIからメッセージを取得してローカルストレージに保存します。

#### 既読/未読の管理

- `markMessageAsRead`関数は、特定のメッセージの既読ステータスを更新します。これにはAPIへのリクエストとローカルの状態の更新が含まれます。

#### ブックマークの管理

- `addBookmark`と`removeBookmark`関数は、ユーザーがメッセージをブックマークに追加または削除するための機能を提供します。

#### 状態管理

- `useState`を使用して、メッセージリスト、ブックマーク、ローディング状態、エラー状態を管理します。

#### コンテキストプロバイダ

- `MessageContext.Provider`は、このコンテキストが提供する値（メッセージリスト、ブックマーク、ローディング状態、エラー状態、各種関数）を子コンポーネントに渡します。

この`MessageContext.js`の設計により、メッセージ関連のデータと操作をアプリケーション全体で共有し、アクセスを容易にします。また、メッセージデータの取得、既読/未読ステータスの管理、ブックマーク機能などが一元化されているため、保守性と可読性が向上します。

`MessageContext.js`の使い方は、ReactのContext APIを利用してアプリケーション全体でメッセージデータと関連機能を共有することにあります。以下のステップでその使い方を説明します。

### 1. MessageContextプロバイダの設定

アプリケーションのトップレベルで`MessageProvider`を配置します。これにより、アプリケーションのどのコンポーネントからもメッセージ関連のデータと機能にアクセスできるようになります。

#### App.js（または同様のトップレベルコンポーネント）

```javascript
import React from 'react';
import { MessageProvider } from './context/MessageContext';
import HomeScreen from './screens/HomeScreen';

const App = () => {
  return (
    <MessageProvider>
      <HomeScreen />
      {/* 他のコンポーネント */}
    </MessageProvider>
  );
};

export default App;
```

### 2. コンテキストの利用

任意のコンポーネントで`useContext`フックを使用して`MessageContext`からデータや機能にアクセスします。

#### 例: メッセージ一覧を表示するコンポーネント

```javascript
import React, { useContext } from 'react';
import { MessageContext } from '../context/MessageContext';

const MessageList = () => {
  const { messages, markMessageAsRead } = useContext(MessageContext);

  return (
    <div>
      {messages.map(message => (
        <div key={message.id} onClick={() => markMessageAsRead(message.id)}>
          {message.content}
        </div>
      ))}
    </div>
  );
};

export default MessageList;
```

### 3. メッセージ操作の実行

`MessageContext`から取得した関数（例: `markMessageAsRead`）を使用して、特定の操作（例: メッセージの既読マーク）を実行します。

#### 例: メッセージを既読にする

上記の`MessageList`コンポーネント内で、メッセージをクリックしたときに`markMessageAsRead`関数を呼び出して、そのメッセージを既読にマークします。

### 4. 状態の更新と反映

`MessageContext`内で状態が更新されると、その状態を使用しているすべてのコンポーネントにその変更が反映されます。これにより、アプリケーション内のデータフローが一貫性を保ちながら動作します。

このように`MessageContext`を使用することで、メッセージ関連のデータと機能をアプリケーション全体で簡単に共有し、管理することができます。また、コンテキストを使用することで、プロップドリリング（複数のコンポーネントを通じてデータを渡すこと）の問題を避け、より整理されたコードベースを維持できます。



### AuthContext.js の詳細設計

`AuthContext.js`は、ユーザー認証に関連するデータと機能をアプリケーション全体で管理するための設計です。ユーザーのログイン状態、認証情報の保存、認証プロセスの管理などを担います。

#### コメント付きコード

```javascript
import React, { createContext, useState, useEffect } from 'react';
import { AuthService } from '../services/AuthService'; // 認証関連のサービス

// 認証コンテキストの作成
export const AuthContext = createContext();

// 認証プロバイダコンポーネント
export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null); // ログインユーザーの状態
  const [loading, setLoading] = useState(true); // 認証プロセス中かどうか

  // ユーザーのログインを処理する関数
  const signIn = async (credentials) => {
    try {
      const loggedInUser = await AuthService.signIn(credentials);
      setUser(loggedInUser);
    } catch (error) {
      // エラー処理
    }
  };

  // ユーザーのログアウトを処理する関数
  const signOut = async () => {
    try {
      await AuthService.signOut();
      setUser(null);
    } catch (error) {
      // エラー処理
    }
  };

  // 認証状態の初期化
  useEffect(() => {
    const initializeAuth = async () => {
      try {
        const currentUser = await AuthService.getCurrentUser();
        setUser(currentUser);
      } catch (error) {
        // エラー処理
      } finally {
        setLoading(false);
      }
    };

    initializeAuth();
  }, []);

  return (
    <AuthContext.Provider value={{ user, loading, signIn, signOut }}>
      {children}
    </AuthContext.Provider>
  );
};
```

### 解説

#### コンテキストの作成

- `createContext`を使用して`AuthContext`を作成し、アプリケーション全体でユーザー認証に関連する状態と機能を共有できるようにします。

#### ステート管理

- `useState`を使用して、現在のユーザー(`user`)と認証プロセス中かどうかを示す`loading`状態を管理します。

#### 認証機能

- `signIn`: ユーザーがログインする際に呼び出され、認証サービスを通じてユーザー認証を行い、`user`ステートを更新します。
- `signOut`: ユーザーがログアウトする際に呼び出され、認証サービスを通じてログアウト処理を行い、`user`ステートをnullに設定します。

#### 認証状態の初期化

- `useEffect`を使用して、コンポーネントのマウント時に現在のユーザー認証状態を初期化します。これにより、アプリケーションの起動時にユーザーがログインしているかどうかを確認し、状態を更新します。

#### コンテキストプロバイダ

- `AuthContext.Provider`は、`AuthContext`が提供する値（ユーザー情報、ログイン/ログアウト機能）を子コンポーネントに渡します。これにより、これらの値や機能にアプリケーションのどの部分からでもアクセスできます。

この設計により、`AuthContext`はアプリケーション全体のユーザー認証関連のデータと操作

を効果的に管理し、各コンポーネントが認証状態に基づいて適切に振る舞えるようにします。また、認証ロジックが一箇所に集中されるため、保守性と可読性が向上します。

`AuthContext.js`の使用方法は、アプリケーション全体でユーザー認証情報と関連機能を共有するために、ReactのContext APIを利用することにあります。以下にその使用方法を詳しく説明します。

### 1. AuthContextプロバイダの設定

アプリケーションのトップレベルコンポーネント（通常は`App.js`）で`AuthProvider`を配置します。これにより、アプリケーションのどの部分からも認証に関連する情報や機能にアクセスできます。

#### App.js（または同様のトップレベルコンポーネント）

```javascript
import React from 'react';
import { AuthProvider } from './context/AuthContext';
import MainComponent from './components/MainComponent';

const App = () => {
  return (
    <AuthProvider>
      <MainComponent />
      {/* 他のコンポーネント */}
    </AuthProvider>
  );
};

export default App;
```

### 2. コンテキストの利用

任意のコンポーネントで`useContext`フックを使用して`AuthContext`からデータや機能にアクセスします。

#### 例: ログインフォームコンポーネント

```javascript
import React, { useContext, useState } from 'react';
import { AuthContext } from '../context/AuthContext';

const LoginForm = () => {
  const [credentials, setCredentials] = useState({ username: '', password: '' });
  const { signIn } = useContext(AuthContext);

  const handleLogin = () => {
    signIn(credentials);
  };

  // フォームのレンダリングとユーザー入力の処理
  // ...

  return (
    <form onSubmit={handleLogin}>
      {/* ユーザー名とパスワードの入力フィールド */}
      {/* ログインボタン */}
    </form>
  );
};

export default LoginForm;
```

### 3. 認証操作の実行

`AuthContext`から取得した関数（例: `signIn`）を使用して、特定の認証操作（例: ユーザーログイン）を実行します。

#### 例: ユーザーをログインさせる

上記の`LoginForm`コンポーネント内で、フォームを送信したときに`signIn`関数を呼び出して、ユーザーをログインさせます。

### 4. 認証状態の更新と反映

`AuthContext`内で認証状態が更新されると、その状態を使用しているすべてのコンポーネントにその変更が反映されます。例えば、ユーザーがログインすると、ユーザー情報がコンテキストにセットされ、ログインに関連するUIが更新されます。

`AuthContext`を使用することで、認証情報と関連機能をアプリケーション全体で簡単に共有し、管理することができます。また、この方法により、認証ロジックが一箇所に集中され、コードの再利用性が向上し、より整理されたコードベースを維持できます。

### ApiService.js の詳細設計

`ApiService.js`は、アプリケーションとバックエンドサーバー間の通信を担当するサービスです。このサービスは、メッセージの取得、メッセージの既読/未読ステータスの更新、および他のAPI関連の機能を提供します。

#### コメント付きコード

```javascript
// ApiService.js
import { API_ENDPOINT } from '../utils/Constants';

export class ApiService {
  // メッセージを取得する関数
  static async fetchMessages() {
    try {
      const response = await fetch(`${API_ENDPOINT}/messages`);
      if (!response.ok) {
        throw new Error('Failed to fetch messages');
      }
      return await response.json();
    } catch (error) {
      console.error('Error fetching messages:', error);
      throw error;
    }
  }

  // メッセージの既読ステータスを更新する関数
  static async updateMessageStatus(messageId, isRead) {
    try {
      const response = await fetch(`${API_ENDPOINT}/messages/${messageId}/status`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ isRead }),
      });
      if (!response.ok) {
        throw new Error('Failed to update message status');
      }
      return await response.json();
    } catch (error) {
      console.error('Error updating message status:', error);
      throw error;
    }
  }

  // その他のAPI関連関数...
}
```

### 解説

#### API通信の基本

- `ApiService`は、アプリケーションとバックエンドサーバーとの通信を中心とした機能を提供します。ここではJavaScriptの`fetch` APIを使用してHTTPリクエストを行っています。

#### メッセージの取得

- `fetchMessages`: この関数は、指定されたエンドポイント（`API_ENDPOINT`）からメッセージの一覧を取得します。サーバーからのレスポンスが適切でない場合はエラーを投げます。

#### メッセージステータスの更新

- `updateMessageStatus`: 特定のメッセージの既読/未読ステータスを更新するための関数です。これはHTTP POSTリクエストを使って、サーバーにメッセージIDと新しいステータスを送信します。

#### エラーハンドリング

- 各APIリクエストにはエラーハンドリングが含まれており、失敗した場合は適切なエラーメッセージをログに出力し、エラーを投げます。これにより、呼び出し元で適切なエラー処理が可能になります。

この`ApiService.js`の設計により、アプリケーションの他の部分はサーバーとの通信を簡単に行え、データの取得や更新などを効率的に実行できます。また、API通信のロジックが一箇所に集中されるため、保守性と可読性が向上します。

### StorageService.js の詳細設計

`StorageService.js`は、アプリケーションのローカルストレージとのやり取りを管理するサービスです。このサービスは、メッセージやブックマークなどのデータをローカルストレージに保存し、必要に応じて取得する機能を提供します。

#### コメント付きコード

```javascript
// StorageService.js
import AsyncStorage from '@react-native-async-storage/async-storage';

export class StorageService {
  // メッセージをローカルストレージに保存する関数
  static async saveMessages(messages) {
    try {
      const jsonValue = JSON.stringify(messages);
      await AsyncStorage.setItem('messages', jsonValue);
    } catch (error) {
      console.error('Error saving messages to storage:', error);
      throw error;
    }
  }

  // ローカルストレージからメッセージを取得する関数
  static async getMessages() {
    try {
      const jsonValue = await AsyncStorage.getItem('messages');
      return jsonValue != null ? JSON.parse(jsonValue) : null;
    } catch (error) {
      console.error('Error retrieving messages from storage:', error);
      throw error;
    }
  }

  // ブックマークをローカルストレージに保存する関数
  static async saveBookmarks(bookmarks) {
    try {
      const jsonValue = JSON.stringify(bookmarks);
      await AsyncStorage.setItem('bookmarks', jsonValue);
    } catch (error) {
      console.error('Error saving bookmarks to storage:', error);
      throw error;
    }
  }

  // ローカルストレージからブックマークを取得する関数
  static async getBookmarks() {
    try {
      const jsonValue = await AsyncStorage.getItem('bookmarks');
      return jsonValue != null ? JSON.parse(jsonValue) : null;
    } catch (error) {
      console.error('Error retrieving bookmarks from storage:', error);
      throw error;
    }
  }

  // その他のストレージ操作関数...
}
```

### 解説

#### ローカルストレージの利用

- `StorageService`はReact Nativeの`AsyncStorage`を使用して、アプリケーションのデータをローカルストレージに保存し、取得します。`AsyncStorage`はキー値ストアであり、単純な文字列データの保存に適しています。

#### メッセージとブックマークの管理

- `saveMessages`と`getMessages`: これらの関数は、メッセージをローカルストレージに保存し、必要に応じて取得するために使用されます。データはJSON形式で保存されます。
- `saveBookmarks`と`getBookmarks`: ユーザーがブックマークしたメッセージの保存と取得を担当します。

#### エラーハンドリング

- 各関数にはエラーハンドリングが実装されており、ストレージ操作中に発生したエラーを適切に処理します。これにより、呼び出し元でエラーをキャッチして適切な対応を取ることができます。

`StorageService.js`の設計により、アプリケーションはユーザーのデータを効率的にローカルストレージに保存し、オフライン時でもこれらのデータにアクセスできるようになります。また、データのローカル保存と取得のロジックが一箇所に集中することで、保守性と可読性が向上します。

