---
lab:
  title: ラボ 2 -Azure SQL Database をプロビジョニングする
  module: Plan and Implement Data Platform Resources
---

# Azure SQL Database をプロビジョニングする

**推定所要時間: 40 分**

受講者は、Virtual Network エンドポイントを使用して Azure SQL Database をデプロイするために必要な基本のリソースを構成します。 SQL データベースへの接続は、ラボ VM (使用可能な場合) またはローカル コンピューターのセットアップから SQL Server Management Studio を使用して検証されます。

AdventureWorks のデータベース管理者は、Virtual Network エンドポイントを含む新しい SQL データベースをセットアップして、デプロイのセキュリティを強化および簡略化します。 SQL Server Management Studioを使用して、データ クエリと結果保持のための SQL Notebook の使用を評価します。

## Azure portal に移動する

1. 提供されていればラボ仮想マシンで、そうでなければローカル コンピューターで、ブラウザ ウィンドウを開きます。

1. [https://portal.azure.com](https://portal.azure.com/) から Azure ポータルに移動します。 Azure アカウント、または提供された資格情報がある場合はそれを使用して、Azure portal にログインします。

1. Azure portal の上部にある検索ボックスで*リソース グループ*を検索し、オプションの一覧から**リソース グループ**を選択します。

1. **[リソース グループ]** ページが表示されている場合は、*contoso-rg* で始まるリソース グループを選択します。 このリソース グループが存在しない場合は、ローカル リージョンに *contoso-rg* という名前の新しいリソース グループを作成するか、既存のリソース グループを使用してそのリージョンをメモします。

## 仮想ネットワークを作成します

1. Azure portal ホーム ページで、左側のメニューを選択します。  

1. 左側のナビゲーション ペインで、 **[仮想ネットワーク]** をクリックします  

1. **[+ 作成]** をクリックして **[仮想ネットワークの作成]** ページを開きます。 **[基本]** タブで次の情報を入力します。

    - **サブスクリプション:** &lt;自分のサブスクリプション&gt;
    - **リソース グループ**: *DP300* または以前に選択したリソース グループで始まる
    - **名前:** lab02-vnet
    - **リージョン:** リソース グループが作成されたのと同じリージョンを選択する

1. **[確認と作成]** を選択し、新しい仮想ネットワークの設定を確認して、 **[作成]** を選択します。

## Azure Portal で Azure SQL Database をプロビジョニングする

1. Azure portal の上部にある検索ボックスで *SQL データベース* を検索し、オプションの一覧から **SQL データベース**を選択します。

1. **[SQL データベース]** ブレードで、**[+ 作成]** を選択します。

1. **[SQL Database の作成]** ページの **[Basics]** タブで次のオプションを選択し、 **[次へ: ネットワーク]** を選択します。

    - **サブスクリプション** &lt;お使いのサブスクリプション&gt;
    - **リソース グループ**: *DP300* または以前に選択したリソース グループで始まる
    - **データベース名:** AdventureWorksLT
    - **サーバー:** **[新規作成]** リンクを選択します。 **[SQL Database サーバーの作成]** ページが表示されます。 サーバーの詳細を次のように指定します。
        - **サーバー名:** dp300-lab-&lt;イニシャル (小文字)&gt; 、必要に応じて 5 桁のランダムな数字 (サーバー名はグローバルに一意である必要がある)
        - **リージョン:** &lt;リソース グループ用に選択したリージョンと同じローカル リージョン (それ以外は、失敗する可能性がある)&gt;
        - **認証方法:** SQL 認証を使用する
        - **サーバー管理者のログイン:** dp300admin
        - **パスワード:** 複雑なパスワードを選択してメモしておきます
        - **パスワードの確認:** 以前に選択したパスワードと同じパスワードを選択します
    - **[OK]** を選択し、**[SQL Database の作成]** ページに戻ります。
    - **[エラスティック プールを使用しますか?]** **[いいえ]** に設定します。
    - **ワークロード環境**: 開発
    - **[コンピューティングとストレージ]** オプションで、**[データベースの構成]** リンクを選択します。 **[構成]** ページの **[サービス レベル]** ドロップダウンで、 **[基本]**、**[適用]** の順に選択します。

1. **[バックアップ ストレージの冗長性]** オプションについては、既定値の **[ローカル冗長バックアップ ストレージ]** のままにします。

1. 次に **[次へ: ネットワーク]** を選択します。

1. **[ネットワーク]** タブ の **[ネットワーク接続]** オプションで、 **[プライベート エンドポイント]** ラジオ ボタンを選択します。

1. 次に、 **[プライベート エンドポイント]** オプションの下の **[+ プライベート エンドポイントの追加]** リンクを選択します。

1. 右側ペインの **[プライベート エンドポイントの作成]** を次のように設定します。

    - **サブスクリプション:** &lt;自分のサブスクリプション&gt;
    - **リソース グループ**: *DP300* または以前に選択したリソース グループで始まる
    - **リージョン:** &lt;リソース グループ用に選択したリージョンと同じローカル リージョン (それ以外は、失敗する可能性がある)&gt;
    - **名前:** DP-300-SQL-Endpoint
    - **ターゲット サブリソース:** SqlServer
    - **仮想ネットワーク:** lab02-vnet
    - **サブネット:** lab02-vnet/default (10.x.0.0/24)
    - **プライベート DNS ゾーンと統合する:** はい
    - **プライベート DNS ゾーン:** 既定値のままにする
    - 設定を確認し、**[OK]** を選択します  

1. 新しいエンドポイントが **[プライベート エンドポイント]** の一覧に表示されます。

1. **[次へ: セキュリティ]** 、 **[次へ: 追加設定]** の順に選択します。  

1. **[追加設定]** ページで、 **[既存のデータを使用]** オプションの **[サンプル]** を選択します。 サンプル データベースのポップアップ メッセージが表示される場合は、 **[OK]** を選択します。

1. **[確認および作成]** を選択します。

1. 設定を確認してから **[作成]** を選択します。

1. デプロイが完了したら、**[リソースに移動]** を選択します。

## Azure SQL Database へのアクセスを有効にする

1. **[SQL データベース]** ページで **[概要]** セクションを選択し、上部セクション内のサーバー名のリンクを選択します。

1. [SQL サーバー] ナビゲーション ブレードで、 **[セキュリティ]** セクションの **[ネットワーク]** を選択します。

1. **[パブリック アクセス]** タブで、**[選択されたネットワーク]** を選択します。

1. **[クライアント IPv4 アドレスの追加]** を選択します。 これにより、現在の IP アドレスから SQL Server へのアクセスを許可するファイアウォール規則が追加されます。

1. **[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]** プロパティをオンにします。

1. **[保存]** を選択します。

---

## SQL Server Management Studio で Azure SQL Database に接続する

1. Azure portal の左側のナビゲーション ペインで、**[SQL データベース]** を選択します。 次に、**AdventureWorksLT** データベースを選択します。

1. **[概要]** ページで、**[サーバー名]** の値をコピーします。

1. ラボ仮想マシンが提供されている場合は、そこから SQL Server Management Studio を起動します。提供されていない場合は、自分のローカル コンピューターから起動します。

1. **[サーバーに接続]** ダイアログで、Azure portal からコピーした **[サーバー名]** の値を貼り付けます。

1. **[認証]** ドロップダウンで、**[SQL Server 認証]** を選択します。

1. **Login** フィールドに「**dp300admin**」と入力します。

1. **[パスワード]** フィールドに、SQL Server の作成時に選択したパスワードを入力します。

1. **[接続]** を選択します。

1. SQL Server Management Studio を使用して、Azure SQL Databaseに接続します。 サーバー、**[データベース]** ノードの順に展開し、 *AdventureWorksLT* データベースを表示できます。

## SQL Server Management Studio で Azure SQL Database のクエリを実行する

1. SQL Server Management Studio で、*AdventureWorksLT*データベースを右クリックし、 **[新しいクエリ]** を選択します。

1. クエリ ウィンドウに次の SQL ステートメントを貼り付けます。

    ```sql
    SELECT TOP 10 cust.[CustomerID], 
        cust.[CompanyName], 
        SUM(sohead.[SubTotal]) as OverallOrderSubTotal
    FROM [SalesLT].[Customer] cust
        INNER JOIN [SalesLT].[SalesOrderHeader] sohead
             ON sohead.[CustomerID] = cust.[CustomerID]
    GROUP BY cust.[CustomerID], cust.[CompanyName]
    ORDER BY [OverallOrderSubTotal] DESC
    ```

1. ツール バーで **[実行]** ボタンを選択して、クエリを実行します。

1. **[結果]** ペインでクエリの結果を確認します。

1. *AdventureWorksLT* データベースを右クリックし、**[新しいクエリ]** を選択します。

1. クエリ ウィンドウに次の SQL ステートメントを貼り付けます。

    ```sql
    SELECT TOP 10 cat.[Name] AS ProductCategory, 
        SUM(detail.[OrderQty]) AS OrderedQuantity
    FROM salesLT.[ProductCategory] cat
        INNER JOIN [SalesLT].[Product] prod
            ON prod.[ProductCategoryID] = cat.[ProductCategoryID]
        INNER JOIN [SalesLT].[SalesOrderDetail] detail
            ON detail.[ProductID] = prod.[ProductID]
    GROUP BY cat.[name]
    ORDER BY [OrderedQuantity] DESC
    ```

1. ツール バーで **[実行]** ボタンを選択して、クエリを実行します。

1. **[結果]** ペインでクエリの結果を確認します。

1. SQL Server Management Studio を閉じます。 変更を保存するかどうかを確認するメッセージが表示されたら、**[いいえ]** を選択します。

---

## リソースをクリーンアップする

仮想マシンを他の目的で使用していない場合は、このラボで作成したリソースをクリーンアップできます。

### リソース グループを削除します

このラボの新しいリソース グループを作成した場合は、リソース グループを削除して、このラボで作成されたすべてのリソースを削除できます。

1. Azure portal で、左側のナビゲーション ペインから **リソース グループ** を選択するか、検索バーで **リソース グループ**を検索して、結果からそのリソース グループを選択します。

1. このラボ用に作成したリソース グループに移動します。 リソース グループには、仮想マシンと、このラボで作成されたその他のリソースが含まれます。

1. トップ メニューから **[リソース グループの削除]** を選択します。

1. **[リソース グループの削除]** ダイアログで、リソース グループの名前を入力して確認した後、**[削除]** を選択します。

1. リソース グループが削除されるのを待ちます。

1. Azure Portal を閉じます。

### ラボ リソースのみを削除する

このラボ用の新しいリソース グループを作成せず、リソース グループとその以前のリソースをそのまま残したい場合は、このラボで作成したリソースを削除できます。

1. Azure portal で、左側のナビゲーション ペインから **リソース グループ** を選択するか、検索バーで **リソース グループ**を検索して、結果からそのリソース グループを選択します。

1. このラボ用に作成したリソース グループに移動します。 リソース グループには、仮想マシンと、このラボで作成されたその他のリソースが含まれます。

1. ラボで以前に指定した SQL Server 名のプレフィックスが付いたすべてのリソースを選択します。 さらに、作成した仮想ネットワークとプライベート DNS ゾーンを選択します。

1. トップ メニューから、**削除**を選択します。

1. **[リソースの削除]** ダイアログで、「**delete**」と入力し、**[削除]** を選択します。

1. **[削除]** をもう一度選択し、リソースの削除を確定します。

1. リソースが削除されるまで待ちます。

1. Azure Portal を閉じます。

---

このラボは以上で完了です。

この演習では、Virtual Network エンドポイントを使用して Azure SQL Database をデプロイする方法について説明しました。 また、SQL Server Management Studio を使用して作成した SQL Database に接続することもできました。
