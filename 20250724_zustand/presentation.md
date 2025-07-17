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

- 更新に必要な状態同士の参照を actions に定義する
- 状態の参照をする関数を一ヶ所に集約する
- useShallowをつかった配列への浅い参照
- Stateの参照は状態が必要な末端のコンポーネントで行う
- StoreのライフサイクルをReactコンポーネントと合わせる
- immer を利用してコードを安全に更新する
- Zodと組み合わせたバリデーション

## xxx

