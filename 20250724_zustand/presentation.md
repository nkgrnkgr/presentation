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

### 1. フォームが動的に変わるケース
```typescript
// 職種選択によってフォームが変わる例
const [userType, setUserType] = useState('');
const [formFields, setFormFields] = useState([]);

// 職種が変わるたびに表示フィールドが動的に変更
useEffect(() => {
  if (userType === 'doctor') {
    setFormFields(['name', 'license', 'specialization']);
  } else if (userType === 'pharmacist') {
    setFormFields(['name', 'license', 'pharmacy']);
  }
}, [userType]);
```

### 2. フォーム初期値を動的に変更
```typescript
// 外部データに基づく初期値の設定
const [userData, setUserData] = useState(null);
const [formData, setFormData] = useState({});

useEffect(() => {
  // API から取得したデータで初期値を設定
  if (userData) {
    setFormData({
      name: userData.name || '',
      email: userData.email || '',
      preferences: userData.preferences || {},
    });
  }
}, [userData]);
```

### 3. 状態同士が相互に依存するケース
```typescript
// 複数の状態が相互に影響し合う例
const [country, setCountry] = useState('');
const [cities, setCities] = useState([]);
const [selectedCity, setSelectedCity] = useState('');
const [districts, setDistricts] = useState([]);

// 国が変わると都市リストが変わる
useEffect(() => {
  if (country) {
    fetchCities(country).then(setCities);
    setSelectedCity(''); // 都市選択をリセット
    setDistricts([]); // 区域リストもリセット
  }
}, [country]);

// 都市が変わると区域リストが変わる
useEffect(() => {
  if (selectedCity) {
    fetchDistricts(selectedCity).then(setDistricts);
  }
}, [selectedCity]);
```

### useState管理の課題
- **副作用の連鎖**: useEffectが複数になり、依存関係が複雑
- **状態の一貫性**: 複数の状態間で整合性を保つのが困難
- **再レンダリング**: 状態変更時の不要な再レンダリングが発生
- **テスト困難**: 複雑な状態変更ロジックのテストが困難

---

## 課題：複雑な入庫ダイアログ

### 入庫ダイアログの画面イメージ
- 薬剤師が医薬品を受領時に使用
- 複数の入力フィールドが相互に依存
- 動的な初期値の設定が必要

### 相互依存する複数の状態
```typescript
// 入庫理由の変更により他のフィールドが影響を受ける
storeStockReason → counterPartyId, feeAmount
supplierType → availableCounterParties
deliveryDate → expirationDate
```

---

## React Hook Formの限界

### 初期値の動的な変更
```typescript
// 外部データに基づく初期値設定が困難
const { reset } = useForm();
useEffect(() => {
  reset(dynamicInitialValues);
}, [externalData]);
```

### 副作用による他フィールドの更新
```typescript
// useEffectによる副作用の連鎖
useEffect(() => {
  if (storeStockReason === 'SOME_VALUE') {
    setValue('counterPartyId', '');
    setValue('feeAmount', 0);
  }
}, [storeStockReason]);
```

---

## React Hook Formの限界（続き）

### 不要な再レンダリング（デモ動画）
- watchによる全体の再レンダリング
- フォーム全体の状態監視による性能劣化

### コンポーネントの肥大化
- 複雑な状態管理ロジックがコンポーネント内に散在
- テストが困難
- メンテナンス性の低下

---

## なぜZustandを選んだか

### ダイアログ内状態の一元管理
- 複雑な状態ロジックをストアに集約
- 副作用を含む状態変更を一箇所で管理
- コンポーネントの責務を明確化

---

## なぜZustandを選んだか（続き）

### チームでの検証プロセス
1. プロトタイプの作成
2. パフォーマンス検証
3. 開発体験の評価
4. チーム内での技術選定会議

**結果：** シンプルさと機能性のバランスが最適

---

## 実装パターン①：Actionsによる副作用の管理

### Before：コンポーネント側での複雑な更新処理
```typescript
const { watch, setValue } = useForm();
const storeStockReason = watch('storeStockReason');

useEffect(() => {
  if (storeStockReason === 'SOME_VALUE') {
    setValue('counterPartyId', '');
    setValue('feeAmount', 0);
  }
}, [storeStockReason]);
```

### After：Store内のactionsに集約
```typescript
const useStoreStockDialogStore = create<StoreStockDialogStore>((set) => ({
  storeStockInfoFormValues: INITIAL_FORM_VALUES,
  actions: {
    updateStoreStockReason: (reason) => {
      set((state) => ({
        ...state,
        storeStockInfoFormValues: {
          ...state.storeStockInfoFormValues,
          storeStockReason: reason,
          // 副作用も同じ場所で管理
          counterPartyId: '',
          feeAmount: 0,
        },
      }));
    },
  },
}));
```

---

## 実装パターン①：メリット

### コード例とメリット
- **一貫性**: 状態変更ロジックが一箇所に集約
- **可読性**: 副作用が明確に表現される
- **テスタビリティ**: ロジックの単体テストが容易
- **デバッグ**: Redux DevToolsで状態変更を追跡可能

---

## 実装パターン②：Selectorsによる疎結合化

### selectorsファイルでの一元管理
```typescript
// selectors.ts
export const selectFeeAmount = (state: StoreStockDialogStore) =>
  state.storeStockInfoFormValues.feeAmount;

export const selectStoreStockReason = (state: StoreStockDialogStore) =>
  state.storeStockInfoFormValues.storeStockReason;

export const selectCounterPartyOptions = (state: StoreStockDialogStore) =>
  state.storeStockInfoFormValues.availableCounterParties?.map(party => ({
    value: party.id,
    label: party.name
  })) || [];
```

### データ構造の変更に強い設計
- ストアの構造変更時はselectors.tsのみ修正
- コンポーネントは影響を受けない

---

## 実装パターン②：コンポーネントの責務を明確化

```typescript
// Component
const FeeAmountInput = () => {
  const feeAmount = useStoreStockDialogStore(selectFeeAmount);
  const { updateFeeAmount } = useStoreStockDialogStore(
    useShallow((state) => state.actions)
  );
  
  return (
    <input 
      value={feeAmount} 
      onChange={(e) => updateFeeAmount(Number(e.target.value))} 
    />
  );
};
```

**メリット：**
- コンポーネントはUIの責務のみ
- 状態ロジックはストアとセレクターに分離
- 再利用性の向上

---

## 実装パターン③：useShallowで再レンダリング最適化

### 医薬品リストの例
```typescript
// ❌ 毎回新しいオブジェクトが作られる
const { updateDrugQuantity, updateExpirationDate } = useStore((state) => state.actions);

// ✅ 参照が同じなら再レンダリングされない
const { updateDrugQuantity, updateExpirationDate } = useStore(
  useShallow((state) => state.actions)
);
```

### 親子コンポーネントでの最適化
```typescript
// 親コンポーネント
const DrugList = () => {
  const drugs = useStoreStockDialogStore(selectDrugList);
  return (
    <div>
      {drugs.map(drug => (
        <DrugItem key={drug.id} drugId={drug.id} />
      ))}
    </div>
  );
};

// 子コンポーネント - 該当する薬剤の変更時のみ再レンダリング
const DrugItem = ({ drugId }) => {
  const drug = useStoreStockDialogStore(
    useCallback((state) => selectDrugById(state, drugId), [drugId])
  );
  // ...
};
```

---

## 実装パターン③：パフォーマンスの改善

### 再レンダリング最適化の効果
- **Before**: フォーム全体の再レンダリング
- **After**: 変更された部分のみ再レンダリング

### 測定結果
- 大規模フォーム（50+ inputs）での性能向上
- レンダリング回数: 70%削減
- ユーザー体験の向上

---

## その他の工夫

### Immerによる安全な状態更新
```typescript
// Immerを使用した直感的な状態更新
const useStoreStockDialogStore = create<StoreStockDialogStore>()(
  devtools(
    immer((set) => ({
      drugs: [],
      actions: {
        updateDrugQuantity: (drugId, quantity) => {
          set((state) => {
            // 不変性を保ちながら直感的な更新
            const drug = state.drugs.find(d => d.id === drugId);
            if (drug) {
              drug.quantity = quantity;
            }
          });
        },
      },
    }))
  )
);
```

### Redux DevToolsでのデバッグ
- アクションの履歴追跡
- 状態の時系列変化を視覚化
- タイムトラベルデバッグ

---

## その他の工夫（続き）

### Zodを使ったバリデーション実装
```typescript
import { z } from 'zod';

const StoreStockFormSchema = z.object({
  drugId: z.string().min(1, '薬剤を選択してください'),
  quantity: z.number().min(1, '数量は1以上で入力してください'),
  expirationDate: z.date().min(new Date(), '有効期限は未来の日付を入力してください'),
});

// ストア内でのバリデーション
const actions = {
  validateAndUpdateForm: (formValues) => {
    const result = StoreStockFormSchema.safeParse(formValues);
    if (!result.success) {
      set((state) => {
        state.errors = result.error.flatten();
      });
      return;
    }
    // 正常な更新処理
  },
};
```

---

## テスト戦略

### Testing Libraryでの結合テスト
```typescript
import { renderHook, act } from '@testing-library/react';
import { useStoreStockDialogStore } from './store';

test('入庫理由の変更時に関連フィールドがリセットされる', () => {
  const { result } = renderHook(() => useStoreStockDialogStore());
  
  // 初期状態設定
  act(() => {
    result.current.actions.updateCounterPartyId('test-id');
    result.current.actions.updateFeeAmount(1000);
  });
  
  // 入庫理由変更
  act(() => {
    result.current.actions.updateStoreStockReason('DIRECT_PURCHASE');
  });
  
  // 関連フィールドがリセットされることを確認
  expect(result.current.storeStockInfoFormValues.counterPartyId).toBe('');
  expect(result.current.storeStockInfoFormValues.feeAmount).toBe(0);
});
```

---

## テスト戦略（続き）

### 複雑な振る舞いの検証
```typescript
test('複数の薬剤の数量変更時の合計金額計算', () => {
  const { result } = renderHook(() => useStoreStockDialogStore());
  
  // 薬剤追加
  act(() => {
    result.current.actions.addDrug({ id: 'drug1', unitPrice: 100 });
    result.current.actions.addDrug({ id: 'drug2', unitPrice: 200 });
  });
  
  // 数量更新
  act(() => {
    result.current.actions.updateDrugQuantity('drug1', 5);
    result.current.actions.updateDrugQuantity('drug2', 3);
  });
  
  // 合計金額の確認
  expect(result.current.totalAmount).toBe(1100); // 100*5 + 200*3
});
```

### チームの自信向上
- 複雑なロジックのテストが容易
- リグレッションテストの充実
- 安心してリファクタリングが可能

---

## 成果・ビフォーアフター

### 再レンダリングの改善（デモ比較）
- **Before**: 入力時の全体再レンダリング
- **After**: 変更部分のみの最適化されたレンダリング

### コードの可読性向上
```typescript
// Before: 分散したロジック
const FormComponent = () => {
  // 100+ lines of state management logic
  useEffect(() => { /* side effect 1 */ }, []);
  useEffect(() => { /* side effect 2 */ }, []);
  useEffect(() => { /* side effect 3 */ }, []);
  // ...
};

// After: 集約されたロジック
const FormComponent = () => {
  const formValues = useStoreStockDialogStore(selectFormValues);
  const { updateFormField } = useStoreStockDialogStore(
    useShallow((state) => state.actions)
  );
  
  // Clean UI logic only
};
```

---

## 成果・ビフォーアフター（続き）

### バグの減少
- **Before**: 状態同期のバグが頻発
- **After**: 中央集権化により同期バグが根本的に解決

### 開発効率の向上
- 新機能追加時間: 40%短縮
- バグ修正時間: 60%短縮
- コードレビュー時間: 30%短縮

---

## 得られた知見

### Zustandの強みが活きるケース
1. **複雑なフォーム**: 相互依存する状態が多い
2. **リアルタイム性**: 状態変更の即座な反映が必要
3. **パフォーマンス重視**: 大量のデータを扱う場面
4. **チーム開発**: 状態管理の標準化が重要

### 導入時の注意点
- 過度な最適化は避ける
- セレクターの適切な粒度設定
- テストしやすい設計を心がける
- チーム全体での設計方針の統一

---

## 得られた知見（続き）

### チーム開発での効果
- **学習コスト**: 新メンバーでも短期間で習得可能
- **コードレビュー**: 状態管理ロジックの理解が容易
- **保守性**: 一貫したパターンでメンテナンス効率向上
- **拡張性**: 新機能追加時の影響範囲が明確

---

## まとめ

### 複雑なフォームでのZustandの有効性
- 状態管理の一元化による保守性向上
- パフォーマンスの最適化
- テスタビリティの向上
- チーム開発の効率化

### 状態管理の設計パターン
- Actionsによる副作用管理
- Selectorsによる疎結合化
- useShallowによる再レンダリング最適化

### テストとの相性の良さ
- 単体テストの容易さ
- 結合テストでの振る舞い検証
- 継続的な品質向上

---

## 質疑応答

### GitHub/ブログ記事へのリンク
- [Zustand公式ドキュメント](https://github.com/pmndrs/zustand)
- [ブログ記事：Zustandを用いた実践的状態管理](https://kakehashi-dev.hatenablog.com/entry/2024/09/10/110000)

### 連絡先
- GitHub: @nkgrnkgr
- Twitter: @nkgrnkgr

**ありがとうございました！**