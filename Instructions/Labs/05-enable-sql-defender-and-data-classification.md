---
lab:
  title: ラボ 5 – Microsoft Defender for SQL とデータ分類を有効にする
  module: Implement a Secure Environment for a Database Service
---

# Microsoft Defender for SQL とデータ分類を有効にする

**推定所要時間:20 分**

学生は、レッスンで得た情報を利用して、Azure portal と AdventureWorks データベース内でセキュリティを構成して実装します。

データベース環境のセキュリティを確保するために、シニア データベース管理者として採用されました。 これらのタスクは Azure SQL Database に焦点を当てます。

## Microsoft Defender for SQL を有効にする

1. ラボの仮想マシンからブラウザー セッションを開始し、[https://portal.azure.com](https://portal.azure.com/) に移動します。 このラボ仮想マシンの **[リソース]** タブで提供されている Azure の **[ユーザー名]** と **[パスワード]** を使用してポータルに接続します。

    ![画像 1](../images/dp-300-module-01-lab-01.png)

1. Azure portal の上部にある検索ボックスで “SQL サーバー” を検索し、オプションの一覧から **SQL サーバー**をクリックします。

    ![自動的に生成されたソーシャル メディアの投稿についての説明のスクリーンショット](../images/dp-300-module-04-lab-1.png)

1. サーバー名 **dp300-lab-XXXXXXXX** を選択すると、詳細ページが表示されます (SQL サーバーに別のリソース グループと場所が割り当てられている場合があります)。

    ![自動的に生成されたソーシャル メディアの投稿についての説明のスクリーンショット](../images/dp-300-module-04-lab-2.png)

1. Azure SQL サーバーのメイン ブレードから、**[セキュリティ]** セクションに移動し、**[Microsoft Defender for Cloud]** を選びます。

    ![Microsoft Defender for Cloud オプションの選択のスクリーンショット](../images/dp-300-module-05-lab-01.png)

    **[Microsoft Defender for Cloud]** ページで、**[Microsoft Defender for SQL を有効にする]** を選びます。

1. Azure Defender for SQL が正常に有効になると、次の通知メッセージが表示されます。

    ![[構成] オプションの選択のスクリーンショット](../images/dp-300-module-05-lab-02_1.png)

1. **[Microsoft Defender for Cloud]** ページで、 **[構成]** リンクを選択します (このオプションを表示するには、ページの更新が必要になる場合があります)

    ![[構成] オプションの選択のスクリーンショット](../images/dp-300-module-05-lab-02.png)

1. **[サーバー設定]** ページで、 **[MICROSOFT DEFENDER FOR SQL]** の下のトグル スイッチが **[オン]** に設定されていることを確認します。

## データ分類を有効にする

1. Azure SQL サーバーのメイン ブレードで、 **[設定]** セクションに移動し、 **[SQL データベース]** を選択し、データベース名を選択します。

    ![AdventureWOrksLT データベースを選択している様子を示すスクリーンショット](../images/dp-300-module-05-lab-04.png)

1. **AdventureWorksLT** データベースのメイン ブレードで、 **[セキュリティ]** セクションに移動し、 **[データの検出と分類]** を選択します。

    ![[データの検出と分類] が表示されたスクリーンショット](../images/dp-300-module-05-lab-05.png)

1. **[データの検出と分類]** ページに、「**現在、SQL Information Protection ポリシーを使用しています。分類の推奨事項を含む列が 15 個見つかりました。** 」という情報メッセージが表示されます。 このリンクを選択します。

    ![分類の推奨事項が表示されたスクリーンショット](../images/dp-300-module-05-lab-06.png)

1. 次の **[データの検出と分類]** 画面で、**[すべて選択]** の横にあるチェック ボックスをオンにして、**[選択した推奨事項を受け入れます]** を選択し、**[保存]** を選択して分類をデータベースに保存します。

    ![[選択した推奨事項を受け入れます] を示すスクリーンショット](../images/dp-300-module-05-lab-07.png)

1. **[データの検出と分類]** 画面に戻り、5 つの異なるテーブルで 15 個の列が正常に分類されたことがわかります。

    ![[選択した推奨事項を受け入れます] を示すスクリーンショット](../images/dp-300-module-05-lab-08.png)

この演習では、Microsoft Defender for SQL を有効にすることで、Azure SQL Database のセキュリティを強化しました。 また、Azure portal の推奨事項に基づいて分類された列を作成しました。
