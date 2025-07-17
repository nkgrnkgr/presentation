---
marp: true
theme: default
paginate: true
backgroundColor: #fff
backgroundImage: url('https://marp.app/assets/hero-background.svg')
header: 'Zustandを用いた実践的状態管理'
footer: '2025/07/24 - Offers Tech Event'
---

# Zustandを用いた実践的状態管理
## 複雑なフォームへの適用事例

---

![bg right:50%](https://avatars.githubusercontent.com/u/17094072?v=4)

## 自己紹介

**株式会社カケハシ**
**生成AI研究開発チーム**
**ソフトウェアエンジニア**
**Nokogiri(@nkgrnkgr)**

---

![bg right:60%](https://cdn.prod.website-files.com/612f503d75b76e04fe50e50f/615e8dbfa2ab7eceed6e4d38_l-mission_img01.jpg)

## 株式会社カケハシ

- 医療体験をしなやかに
- 主に薬局向けの業務システムを提供
- ヘルステックスタートアップ

---

## 今日話すこと

1. useState以上の状態管理が必要なケース
2. Zustandを使った設計・実装プラクティス
3. 得られた知見・課題

---

## useState以上の状態管理が必要なケース

- 複雑な状態間の依存関係
- 散在する状態更新ロジック
- 頻繁な再描画によるパフォーマンス問題

---

![bg top:100%](https://cdn-ak.f.st-hatena.com/images/fotolife/k/kakehashi_dev/20240909/20240909094358.png)

---

## 具体的な課題：AI在庫ダイアログ

- 動的に変わる初期値
- 更新時に他の状態を更新する

---

## Zustandによる解決

- 中央集権的な状態管理
- アクションによる状態更新の集約
- セレクターによる効率的な再描画制御

---

## Zustand 具体的なプラクティス

---

### 1. Actions に状態更新ロジックを集約する

```typescript
const useStoreStockDialogStore = create<StoreStockDialogStore>()(
  immer((set) => ({
    actions: {
      updateStoreStockReason: (reason) => {
        set((state) => {
          // プライマリ状態の更新
          state.storeStockInfoFormValues.storeStockReason = reason;

          // 他の状態への副作用
          state.storeStockInfoFormValues.counterPartyId = INITIAL_FORM_VALUES.counterPartyId;
          state.storeStockInfoFormValues.counterPartyName = INITIAL_FORM_VALUES.counterPartyName;
          state.stockOperationConfig.counterParty = null;
        });
      },
    },
  }))
);
```

---

### 2. セレクターによる効率的な状態参照

```typescript
// selectors.ts
export const selectFeeAmount = (state: StoreStockDialogStore) =>
  state.storeStockInfoFormValues.feeAmount;

// Component
const feeAmount = useStoreStockDialogStore(selectFeeAmount);
```

- 状態の参照を一箇所に集約
- 再描画の最適化
- 型安全性の向上

---

### 3. useShallow を使った配列への浅い参照

```typescript
const selectMedicineIds = (state: StoreStockDialogStore) =>
  state.medicineInfoList.map((i) => i.medicineId);

function MedicineList() {
  const ids = useStoreStockDialogStore(useShallow(selectMedicineIds));
  return (
    <ul>
      {ids.map((id) => (
        <MedicineListItem key={id} medicineId={id} />
      ))}
    </ul>
  );
}
```

- 配列の内容が変わらなければ再描画を防ぐ
- パフォーマンスの向上

---

### 4. 末端コンポーネントでの状態参照

```typescript
function MedicineListItem({ medicineId }: { medicineId: string }) {
  const medicine = useStoreStockDialogStore(
    (state) => state.medicineInfoList.find(m => m.medicineId === medicineId)
  );
  
  return <div>{medicine?.name}</div>;
}
```

- 必要な状態のみを参照
- 不要な再描画を防ぐ

---

### 5. Store のライフサイクル管理

```typescript
// StoreProvider.tsx
export const StoreProvider = ({ children }: { children: ReactNode }) => {
  const storeRef = useRef<StoreApi<StoreStockDialogStore>>();
  
  if (!storeRef.current) {
    storeRef.current = createStoreStockDialogStore();
  }
  
  return (
    <StoreContext.Provider value={storeRef.current}>
      {children}
    </StoreContext.Provider>
  );
};
```

- React コンポーネントのライフサイクルと連動
- メモリリークの防止

---

### 6. immer を利用した安全な更新

```typescript
const useStore = create<Store>()(
  immer((set) => ({
    nested: { value: 0 },
    actions: {
      updateNested: (value) => {
        set((state) => {
          state.nested.value = value; // 直接代入可能
        });
      },
    },
  }))
);
```

- 不変性を保ちながら直感的な更新
- バグの減少

---

### 7. Zod と組み合わせたバリデーション

```typescript
const selectStoreStockInfoErrors = (state: StoreStockDialogStore) => {
  const result = storeStockInfoFormSchema.safeParse(
    state.storeStockInfoFormValues
  );
  return result.success ? null : result.error.flatten();
};

// Component
const errors = useStoreStockDialogStore(selectStoreStockInfoErrors);
```

- 型安全なバリデーション
- リアルタイムエラー表示

---

## 効果：実装による改善

### 1. パフォーマンス向上

- **Before**: 20回以上の再レンダリング
- **After**: 5回程度の再レンダリング

### 2. コードの見通しが良くなった

- 状態更新ロジックが actions に集約
- 散在していた状態管理コードが整理
- 型安全なセレクターによる一貫した状態参照

### 3. 機能拡張に強くなった

- 新しい状態の追加が容易
- 既存の状態との依存関係を明確に表現
- バリデーションルールの追加・変更が簡単

---

![bg left:100%](https://cdn-ak.f.st-hatena.com/images/fotolife/k/kakehashi_dev/20240905/20240905184236.gif)

---

## 得られた知見と課題

### 得られた知見

✅ **パフォーマンス向上**
- 再レンダリング回数の大幅削減
- 複雑な状態管理でもスムーズな動作

✅ **開発体験の向上**
- 状態更新ロジックの集約で保守性向上
- 型安全性とバリデーションの統合

✅ **コードの可読性**
- アクションによる明確な状態更新
- セレクターによる統一的な状態参照

---

### 課題

❌ **学習コスト**
- 適切なセレクターの設計が必要
- useShallow の使いどころの判断

❌ **デバッグの複雑さ**
- 状態の変更フローが分かりにくい場合がある
- Redux DevTools の習熟が必要

❌ **設計の一貫性**
- チーム内でのパターンの統一が重要
- 過度な最適化による複雑化のリスク


