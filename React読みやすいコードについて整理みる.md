# React TypeScript における「読みやすいコード」について整理してみる

現場で「このコードは読みづらいな」と感じるパターンがいくつか見えてきたので、言語化して整理してみたいと思います。
これは技術的な「正解」というよりは、私が「こうなっていると読みやすい・メンテナンスしやすい」と感じるポイントをまとめたものです。

## 1. コンポーネントとロジックの分離

「読みづらい」と感じる最大の要因は、コンポーネントの中にビジネスロジックが大量に書かれているケースです。
1つのファイルを見た時に「このコンポーネントは何をやっているんだ？」と理解するのが難しくなります。

### アプローチ
- **ビジネスロジックは外部ファイル（Libsなど）へ切り出す**
- コンポーネント内では、切り出した関数を `useMemo` や `useCallback` で呼び出すだけの形にする

### サンプルコード

#### 💩 BAD: コンポーネント内にロジックが混在
```tsx
const UserList = ({ users }: { users: User[] }) => {
  // ↓↓↓ ここに数十行〜百行規模のロジックが直接書かれていると想像してください ↓↓↓

  // フィルタリング系の関数をポコポコ呼んでいる
  const processedData = useMemo(() => {
    // 複雑なフィルタリング条件...
    const active = filterActiveUsers(users);
    // 権限チェック...
    const permitted = filterByPermission(active, 'READ_PRIVILEGE');
    // 並び替えロジック...
    const sorted = sortUsersByComplexRules(permitted);
    // 表示用に加工...
     return sorted.map(u => ({
       ...u,
       displayName: formatName(u),
       rank: calculateRank(u.score)
    }));
  }, [users]);

  // 他にもAnalytics送信や副作用が散らばっている
  useEffect(() => {
     if (processedData.length > 0) {
        analytics.track('ViewUserList', { count: processedData.length });
     }
  }, [processedData]);

  return (
    <ul>
      {processedData.map(user => <li key={user.id}>{user.displayName}</li>)}
```

#### ✨ GOOD: ロジックをカスタムフックや関数に分離
```tsx
// useUserLogic.ts などに定義
const useActiveUsers = (users: User[]) => {
  return useMemo(() => getActiveUsersRanked(users), [users]);
};

// Component
const UserList = ({ users }: { users: User[] }) => {
  // 呼び出すだけ！何をしているか一目瞭然
  const activeUsers = useActiveUsers(users);

  return (
    <ul>
      {activeUsers.map(user => <li key={user.id}>{user.name} ({user.rank})</li>)}
    </ul>
  );
};
```

### メリット
- コンポーネントの見通しが良くなる（コード量が数行で済むことも）
- ロジック単体でテストが可能になる（テスタビリティの向上）

## 2. コンポーネントの適切なサイズ（行数）

ファイルが長すぎると、全体を把握するだけで疲れてしまいます。

### 目安
- **多くても 100行〜150行**
- 長くても 200行程度に収める

### サンプルコード

#### 💩 BAD: 巨大なコンポーネント
```tsx
const ProductPage = () => {
  // ... 50行にわたるState定義 ...
  // ... 100行にわたるイベントハンドラとuseEffect ...

  return (
    <div>
      <header>
        {/* ... 50行のヘッダー実装 ... */}
      </header>
      <main>
        {/* ... 100行のメインコンテンツ実装 ... */}
      </main>
      <dialog>
         {/* ... 30行のダイアログ実装 ... */}
      </dialog>
    </div>
  );
};
```

#### ✨ GOOD: 責務ごとにコンポーネントを分割
```tsx
const ProductPage = () => {
  const { product, isLoading } = useProductData();

  if (isLoading) return <Loading />;

  return (
    <div>
      <ProductHeader info={product.info} />
      <ProductMain content={product.content} />
      <PurchaseDialog price={product.price} />
    </div>
  );
};
```

### メリット
- 影響範囲が見えやすくなり、不具合修正や機能追加が容易になる

## 3. アンチパターンを避ける（特に useEffect）

React Hooks、特に `useEffect` の使い方は可読性とバグの温床になりがちです。

### 避けるべき実装
- `useEffect` 内で依存配列を指定し、連鎖的にStateを更新する処理
  - データの流れが追いづらくなります。
  - 基本的にはイベントハンドラ（`onChange` など）で処理を行うべきです。

### 推奨する実装
- **計算で導出できる値は `useMemo` を使う**
  - 例：ユーザー名の加工などはStateにせず、レンダリング時に計算（メモ化）する。
- 参考：[You Might Not Need useEffect](https://react.dev/learn/you-might-not-need-an-effect)

### サンプルコード

#### 💩 BAD: useEffectでStateを同期しようとする
```tsx
const UserProfile = ({ firstName, lastName }: Props) => {
  const [fullName, setFullName] = useState('');

  // Propsが変わるたびに再レンダリング→Effect実行→State更新→再レンダリング...
  useEffect(() => {
    setFullName(`${firstName} ${lastName}`);
  }, [firstName, lastName]);

  return <div>{fullName}</div>;
};
```

#### ✨ GOOD: レンダリング中に計算（必要ならuseMemo）
```tsx
const UserProfile = ({ firstName, lastName }: Props) => {
  // Stateを使わず、レンダリング時に計算する
  const fullName = `${firstName} ${lastName}`;

  return <div>{fullName}</div>;
};
```

## 4. レンダリング部分（return 文）の簡潔化

`return` の中（JSX）が複雑だと、UIの構造が直感的に分かりません。

### ポイント
- **JSX内に複雑なロジックを書かない**
  - `map` や `if` のネストで20行以上になるようなブロックは、別のコンポーネントに切り出す。
  - 上位コンポーネントは「どのコンポーネントをどの順番で表示するか」だけに集中する。
- **適切なコメントと命名**
  - 「ここは画面の○○の部分」と分かるコメントを残す。
  - 全員が直感的に理解できる命名を心がける。

### サンプルコード

#### 💩 BAD: JSX内にロジックとネストが埋め込まれている
```tsx
return (
  <div className="page-container">
    <Header user={user} />
    {/* ヘッダーの下にも直接ロジックでバナー出し分けなどを書いている */}
    {showBanner && !isMobile && (
      <div className="banner">Special Campaign!</div>
    )}

    {isLoading ? (
      <Spinner />
    ) : (
      <div className="main-content">
        {/* サイドナビの表示ロジックもここに混入 */}
        <aside>
           {categories.map(cat => (
             cat.isActive ? <div key={cat.id}>{cat.name}</div> : null
           ))}
        </aside>

        <main>
          <ul className="item-list">
            {items.map(item => {
              // mapの中で変数定義や複雑な分岐を行うと可読性が著しく低下する
              if (item.isHidden) return null;
              const statusLabel = item.stock > 0 ? 'In Stock' : 'Out of Stock';

              return (
                 <li key={item.id} onClick={() => handleClick(item)}>
                   <div className="item-header">
                     <h3>{item.title}</h3>
                     <span className="status">{statusLabel}</span>
                   </div>

                   {/* さらにネストされたmap繰り返し */}
                   {item.tags.length > 0 && (
                     <div className="tags">
                       {item.tags.map(tag => (
                         <span key={tag} className="tag">{tag}</span>
                       ))}
                     </div>
                   )}

                   {/* 条件付きレンダリングの塊 */}
                   {item.hasDetails && (
                     <div className="details">
                       <p>{item.description}</p>
                       {item.relatedUrl && <a href={item.relatedUrl}>Link</a>}
                     </div>
                   )}
                 </li>
              );
            })}
          </ul>
        </main>
      </div>
    )}
    <Footer />
  </div>
);
```

#### ✨ GOOD: 構成要素ごとにコンポーネント化
```tsx
// 親コンポーネント：配置に集中できる
if (isLoading) return <Spinner />;

return (
  <div className="page-container">
    <Header user={user} />
    <CampaignBanner show={showBanner && !isMobile} />

    <div className="main-content">
      {/* サイドバーやリストも独立させる */}
      <CategorySidebar categories={categories} />
      <main>
        <ul className="item-list">
          {items.map(item => (
            <ListItem key={item.id} item={item} onClick={handleClick} />
          ))}
        </ul>
      </main>
    </div>
    <Footer />
  </div>
);

// 子コンポーネント：さらに内部の部品（Header, Tags, Details）を分割
const ListItem = ({ item, onClick }: ListItemProps) => {
  if (item.isHidden) return null;

  return (
    <li onClick={() => onClick(item)}>
      <ItemHeader title={item.title} stock={item.stock} />
      <ItemTags tags={item.tags} />
      {item.hasDetails && (
        <ItemDetails description={item.description} url={item.relatedUrl} />
      )}
    </li>
  );
};
```

## 5. オーケストレーションコンポーネント（単一責任）

トップレベルや親となるコンポーネントは「オーケストラ」の指揮者のような役割に徹するのが理想だと考えています。

### 役割
- APIの呼び出し（`useQuery` 等）
- 定数やメモ化された値の準備
- 子コンポーネントの配置
- **ロジックの詳細は書かない**

詳細な計算や条件分岐をここに書くと一気に読みづらくなります。

### サンプルコード

#### 💩 BAD: データ取得と表示ロジックの混在
```tsx
const UserDashboard = () => {
  const { data: userData } = useSWR('/api/user');
  const { data: postsData } = useSWR('/api/posts');

  // ↓↓↓ ここに数百行規模のデータ加工ロジックが書かれていると想像してください ↓↓↓
  // ...

  // アクティブユーザー判定や権限計算など、本来切り出すべき関数が並ぶ
  const userStatus = useMemo(() => {
     if (!userData) return null;
     const score = calculateUserScore(userData);
     const rank = determineUserRank(score);
     const badges = getUserBadges(userData.history);
     // ... ごにょごにょとした計算が続く ...
     return { score, rank, badges };
  }, [userData]);

  // 推奨コンテンツの選定ロジックなども混ざっている
  const recommendedPosts = useMemo(() => {
     if (!postsData || !userData) return [];
     const filtered = filterPostsByInterests(postsData, userData.interests);
     const sorted = sortPostsByDate(filtered);
     // ... さらにごにょごにょ ...
     return sorted.slice(0, 5);
  }, [postsData, userData]);

  if (!userData) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>Welcome, {userData.name}</h1>
      <div className="stats">
         {/* ...細かい表示実装... */}
         <span>Rank: {userStatus?.rank}</span>
      </div>
      <div className="posts">
         {recommendedPosts.map(p => <div key={p.id}>{p.title}</div>)}
      </div>
    </div>
  );
};
```

#### ✨ GOOD: コンテナとプレゼンテーションの分離
```tsx
// オーケストレーション（Container）
const UserDashboardContainer = () => {
  const { data, error, isLoading } = useUserQuery();

  if (isLoading) return <Loading />;
  if (error) return <ErrorFallback error={error} />;

  // データの準備ができたら、表示専用コンポーネントに渡すだけ
  return <UserDashboardUI user={data} />;
};
```

## 6. 型ガードとデータの境界（null/undefined考慮）

`useQuery` などで取得したデータに含まれる `undefined` や `null` の扱いについて。

### アプローチ
- **上位コンポーネントでデータ有無を判定する**
  - データがない場合（null/undefined/loading）は、早期リターンや専用の表示（NoMatchなど）を行う。
- **下位コンポーネントは「データがある前提」で作る**
  - これにより、下位コンポーネント内で不要な `?` や `if (data)` のチェックを排除でき、コードがシンプルになる。

### サンプルコード

#### 💩 BAD: 下位コンポーネントで毎回 null チェック
```tsx
// Propsの型が undefined 許容になっている
const UserInfo = ({ user }: { user?: User | undefined }) => {
  // 毎回チェックしないといけない
  if (!user) return null;

  return <div>{user.name}</div>;
};
```

#### ✨ GOOD: 親でガードし、子はデータがある前提
```tsx
// 親コンポーネント
const Parent = () => {
  const { user } = useUser();

  // ここでガードする！
  if (!user) return <Loading />;

  return <UserInfo user={user} />;
};

// 子コンポーネント (Propsは User 型で確定)
const UserInfo = ({ user }: { user: User }) => {
  return <div>{user.name}</div>;
};
```

---

## まとめ

今回紹介したアプローチの根底にあるのは、「**適切な単位で分割し、責務を明確にする**」という思想です。
巨大なコンポーネントを解体し、ロジックを隔離し、UIを階層化することで、コードは見通しが良くなります。

しかし、これは「銀の弾丸」ではありません。
「細かく分割しすぎると、逆にファイルを行き来するコストが増えて読みづらい」という意見もあるでしょう。それは至極もっともです。過度な抽象化やマイクロコンポーネント化（Over-engineering）は、かえって全体の把握を困難にし、修正時の工数を増大させるリスクも孕んでいます。

大切なのは、思考停止で分割することではなく、**プロジェクトの規模やチームのスキルセットに応じた「ちょうどいい塩梅」を見つけること**です。
分割による「見通しの良さ（凝集度の向上）」と、ファイル分散による「認知負荷（結合度の管理）」のトレードオフを理解した上で、チームにとって最適なバランスを選択する必要があります。個人開発であれば、自分が一番気持ちよく開発できる分割単位を探求するのも良いでしょう。

今回提示したのは、あくまで今の私が現場で感じている「読みやすいコード」の一例に過ぎません。
この記事が、あなたやあなたのチームにとっての「ベストプラクティス」を議論するきっかけになれば幸いです。
