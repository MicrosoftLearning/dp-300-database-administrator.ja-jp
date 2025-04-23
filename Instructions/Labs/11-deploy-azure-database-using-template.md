---
lab:
  title: 'ラボ 11: Azure Resource Manager テンプレートを使用して Azure SQL Database をデプロイする'
  module: Automate database tasks for Azure SQL
---

# テンプレートから Azure SQL Database をデプロイする

**推定所要時間: 15 分**

あなたはデータベース管理の日常業務を自動化するために、シニア データ エンジニアとして採用されています。 この自動化はピーク時のパフォーマンスで AdventureWorks のデータベースの稼働を確実に続けるうえで役に立つだけでなく、特定の条件に基づいてアラートを生成することもできるようにします。 AdventureWorks では、サービスとしてのインフラストラクチャ (IaaS) とサービスとしてのプラットフォーム (PaaS) の両方のオファリングで SQL Server が使用されています。

## Azure Resource Manager テンプレートを探索する

1. Microsoft Edge で新しいタブを開き、GitHub リポジトリ内の次のパスに移動します。ここには、SQL Database リソースをデプロイするための ARM テンプレートが含まれています

    ```url
    https://github.com/Azure/azure-quickstart-templates/tree/master/quickstarts/microsoft.sql/sql-database
    ```

1. **azuredeploy.json** を右クリックし、 **[新しいタブでリンクを開く]** を選択すると、次のような ARM テンプレートが表示されます。

    ```JSON
    {
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "serverName": {
        "type": "string",
        "defaultValue": "[uniqueString('sql', resourceGroup().id)]",
        "metadata": {
            "description": "The name of the SQL logical server."
        }
        },
        "sqlDBName": {
        "type": "string",
        "defaultValue": "SampleDB",
        "metadata": {
            "description": "The name of the SQL Database."
        }
        },
        "location": {
        "type": "string",
        "defaultValue": "[resourceGroup().location]",
        "metadata": {
            "description": "Location for all resources."
        }
        },
        "administratorLogin": {
        "type": "string",
        "metadata": {
            "description": "The administrator username of the SQL logical server."
        }
        },
        "administratorLoginPassword": {
        "type": "securestring",
        "metadata": {
            "description": "The administrator password of the SQL logical server."
        }
        }
    },
    "variables": {},
    "resources": [
        {
        "type": "Microsoft.Sql/servers",
        "apiVersion": "2020-02-02-preview",
        "name": "[parameters('serverName')]",
        "location": "[parameters('location')]",
        "properties": {
            "administratorLogin": "[parameters('administratorLogin')]",
            "administratorLoginPassword": "[parameters('administratorLoginPassword')]"
        },
        "resources": [
            {
            "type": "databases",
            "apiVersion": "2020-08-01-preview",
            "name": "[parameters('sqlDBName')]",
            "location": "[parameters('location')]",
            "sku": {
                "name": "Standard",
                "tier": "Standard"
            },
            "dependsOn": [
                "[resourceId('Microsoft.Sql/servers', concat(parameters('serverName')))]"
            ]
            }
        ]
        }
    ]
    }
    ```

1. JSON プロパティを確認して観察します。

1. **[azuredeploy.json]** タブを閉じ、**sql-database** GitHub フォルダーを含むタブに戻ります。 下にスクロールし、 **[Azure にデプロイ]** を選択します。

    ![[Azure にデプロイ] ボタン](../images/dp-300-module-11-lab-01.png)

1. Azure portal で **[SQL Server とデータベースの作成]** クイックスタート テンプレート ページが開きます。リソースの詳細の一部は ARM テンプレートから入力されます。 空白のフィールドに次の情報を入力します。

    - **リソース グループ:** *contoso-rg* で始まります
    - **SQL 管理者ログイン:** labadmin
    - **SQL 管理者ログインのパスワード:** &lt;強力なパスワードを入力します&gt;

1. **[確認と作成]** を選択し、次に **[作成]** を選択します。 デプロイには 5 分程度かかります。

    ![画像 2](../images/dp-300-module-11-lab-02.png)

1. デプロイが完了したら、**[リソース グループに移動]** を選択します。 Azure リソース グループに移動します。これには、デプロイによって作成されたランダムな名前の **SQL Server** リソースが含まれています。

    ![図 3](../images/dp-300-module-11-lab-03.png)

---

## リソースをクリーンアップする

Azure SQL Server を他の目的で使用していない場合は、このラボで作成したリソースをクリーンアップできます。

### リソース グループを削除します

このラボの新しいリソース グループを作成した場合は、リソース グループを削除して、このラボで作成されたすべてのリソースを削除できます。

1. Azure portal で、左側のナビゲーション ペインから **リソース グループ** を選択するか、検索バーで **リソース グループ**を検索して、結果からそのリソース グループを選択します。

1. このラボ用に作成したリソース グループに移動します。 リソース グループには、Azure SQL Server と、このラボで作成されたその他のリソースが含まれます。

1. トップ メニューから **[リソース グループの削除]** を選択します。

1. **[リソース グループの削除]** ダイヤログで、リソース グループの名前を入力して確認した後、**[削除]** を選択します。

1. リソース グループが削除されるのを待ちます。

1. Azure Portal を閉じます。

### ラボ リソースのみを削除する

このラボ用の新しいリソース グループを作成せず、リソース グループとその以前のリソースをそのまま残したい場合は、このラボで作成したリソースを削除できます。

1. Azure portal で、左側のナビゲーション ペインから **リソース グループ** を選択するか、検索バーで **リソース グループ**を検索して、結果からそのリソース グループを選択します。

1. このラボ用に作成したリソース グループに移動します。 リソース グループには、Azure SQL Server と、このラボで作成されたその他のリソースが含まれます。

1. ラボで以前に指定した SQL Server 名のプレフィックスが付いたすべてのリソースを選択します。

1. トップ メニューから、**削除**を選択します。

1. **[リソースの削除]** ダイアログで、「**delete**」と入力し、**[削除]** を選択します。

1. **[削除]** をもう一度選択し、リソースの削除を確定します。

1. リソースが削除されるまで待ちます。

1. Azure Portal を閉じます。

---

このラボは以上で完了です。

ここでは、Azure Resource Manager テンプレート リンクを 1 回クリックするだけで、Azure SQL Server とデータベースの両方を簡単に作成できる方法を確認しました。
