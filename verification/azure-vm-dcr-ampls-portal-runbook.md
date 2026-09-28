# Azure VM ログ監視ハンズオン — Azure portal 操作手順書

![Azure VM、閉域閲覧端末、Private Endpoint、AMPLS、DCE、Log Analytics、ログ検索アラートの構成図](../assets/azure-vm-dcr-ampls-private-only.png)

- 対象: Windows Azure VM 1 台 / Azure Monitor Agent (AMA) / Data Collection Rule (DCR) / Log Analytics / Azure Monitor Private Link Scope (AMPLS)
- 状態: 実施前の手順書。
  Azure 環境へのデプロイ、疎通試験、アラート発火試験は未実施。
- 方針: 既存の共有資産がある場合は作り直さない。
  各画面の日本語名は表示差があるため英語名も併記する。
  **作成・変更操作は対象環境の承認取得後に行う。**

## 1. 今回の環境と到達目標

Windows VM の `Application` イベント ログから Error を DCR で選択し、AMA で Log Analytics ワークスペースの `Event` テーブルへ送る。
AMA の `Heartbeat`、任意の CPU `Perf`、KQL、Workbook、Azure ダッシュボード、ログ検索アラートの通知を確認する。
通信面では AMA の DCR 構成取得、ログ取り込み、閲覧端末からのクエリをそれぞれ検証する。
AMPLS のクエリとインジェストは `Private Only` とし、最後に今回の Log Analytics ワークスペースの公開クエリと公開取り込みを無効化する。
**Prometheus 向けの Azure Monitor ワークスペースは作らない。**[^1][^2]

**DCR は転送経路ではなく、VM に適用する収集内容と送信先の定義。**
VM に DCR と DCE の関連付けを作り、AMPLS には Log Analytics ワークスペースと DCE を登録する。
今回の Windows イベントを Log Analytics に送るために、DCR のインジェスト送信先として DCE を選択する必要はない。
DCE は AMA の**構成取得**のために使う。[^1][^2][^3]

| 作成・再利用するもの | 本 PoC での用途 | 本書の手順 |
| --- | --- | --- |
| 既存または新規の VNet、VM サブネット、PE サブネット | VM と Private Endpoint の配置 | 2、3 |
| Windows Server VM × 1 | 検証用イベントの生成 | 3 |
| Log Analytics ワークスペース × 1 | `Heartbeat`、`Event`、任意の `Perf` の保存 | 4 |
| DCE × 1 | AMA のプライベートな構成取得 | 5、8 |
| AMPLS × 1 + Private Endpoint × 1 + Private DNS | ワークスペース・DCE のプライベート接続 | 6、7 |
| DCR × 1 + AMA | Application/Error の収集、任意の CPU カウンター | 8 |
| Workbook、ダッシュボード、Action Group、ログ アラート | 表示と発火確認 | 10、11、12 |

### 1.1 使用する値

次の値は例示の仮名である。
実際の操作では承認済みの値へ置換する。

| 項目 | 使用する値 | 備考 |
| --- | --- | --- |
| サブスクリプション / 検証用 RG | `＜subscription＞` / `＜rg＞` | 既存共有リソースは別 RG のままでよい。 |
| リージョン | `＜region＞` | VM、ワークスペース、DCR、DCE は同一にする。PE は VNet と同一リージョン。 |
| VNet / VM 用サブネット / PE 用サブネット | `＜vnet＞` / `＜vm-subnet＞` / `＜pe-subnet＞` | PE サブネットは /27 以上を目安に IP を確保する。[^4] |
| VM / VM の完全なリソース ID | `＜vm＞` / `＜vm-resource-id＞` | VM の **[概要] → [JSON ビュー]** 等から ID を確認する。 |
| ワークスペース / DCE / AMPLS / PE / DCR | `＜law＞` / `＜dce＞` / `＜ampls＞` / `＜pe＞` / `＜dcr＞` | 既存利用時は名前と RG を確認する。 |
| Workbook / ダッシュボード / Action Group / アラート | `＜workbook＞` / `＜dashboard＞` / `＜ag＞` / `＜alert＞` | PoC 専用と判別できる名称にする。 |
| 閉域の閲覧端末 / 非接続の試験端末 | `＜private-client＞` / `＜public-test-client＞` | 非接続試験は許可を受け、業務端末を無断で切替えない。 |
| 通知先 / 作業・切戻し承認者 | `＜承認済み宛先＞` / `＜担当者＞` | 宛先は portal の Action Group にのみ入力する。 |

### 1.2 作業開始のゲート

1. ポータル上部の **[ディレクトリとサブスクリプション]** で対象テナント・サブスクリプションを確認する。
2. VM とリソースの作成・関連付け・拡張機能の追加、ワークスペース設定の変更、VNet / PE / DNS の編集、アラート作成の権限を担当者別に確認する。
   閲覧者には Log Analytics のクエリ権限、イベント生成担当には VM 内の管理者権限が必要。[^1][^8]
3. 既存 VM の AMA / DCR / DCE 関連付け、ワークスペースの公開アクセス設定、共有 AMPLS・Private DNS、Network Security Perimeter (NSP)、ネットワーク ファイアウォール、通知ルールを棚卸しする。
   既存 DCR を portal で編集すると、JSON で追加した未サポートの設定が失われ得るため、PoC 用 DCR は原則新規に作る。[^1][^4]
4. VM と閲覧端末の双方から PE サブネットへの経路、および必要な DNS フォワーダーをネットワーク担当と確認する。
   ポータルのサインインには Entra ID・ARM・ポータル拡張への到達も必要であり、監視 API の Private Link とは別に扱う。[^4]
5. **停止条件:** 共有 DNS 配下に別の AMPLS がある、既存 DCE を置き換えることになる、既存監視への影響が分からない場合、後続の変更は実施せず担当者に構成を確認する。
   共有 DNS では原則同一 AMPLS を利用する。[^3][^5]

## 2. 作業先を確定し、VNet を用意する

### 2.1 RG と既存構成

1. Azure portal の検索バーで **[リソース グループ]** を検索し、`＜rg＞` を開く。
   存在しなければ **[作成] → [サブスクリプション] / [リソース グループ名] / [リージョン] → [確認および作成] → [作成]**。
2. 検索バーから **[仮想マシン]**、**[Log Analytics ワークスペース]**、**[モニター (Monitor)] → [データ収集ルール] / [データ収集エンドポイント] / [プライベート リンク スコープ]** を順に開く。
   既存のものは本書の「作成」を読み飛ばし、設定と関連付けのみ照合する。
3. **[仮想ネットワーク] → `＜vnet＞` → [DNS サーバー] / [サブネット]** で DNS とサブネットを確認する。
   共有環境の DNS 設定を PoC のために直接変更しない。

### 2.2 VNet が無い場合のみ作成

1. **[仮想ネットワーク] → [作成] → [基本]** でサブスクリプション `＜subscription＞`、RG `＜rg＞`、名前 `＜vnet＞`、リージョン `＜region＞` を設定。
2. **[IP アドレス]** でネットワーク担当が割り当てたアドレス空間と、VM 用 `＜vm-subnet＞`、PE 用 `＜pe-subnet＞` を追加する。
   IP レンジは環境固有なので例示値を流用しない。
   Azure Monitor PE は複数 IP を消費し、/27 未満のサブネットを選ばない。[^4]
3. **[セキュリティ] / [DNS]** は組織標準に従い、既存の NSG、ルート、DNS フォワーダー設計を確認して **[確認および作成] → [作成]**。
   PE を適用する NSG / UDR ポリシーは既存設計の承認なしに変更しない。

**完了確認:** VNet の **[サブネット]** に VM と PE 用の二つが存在し、PE 用アドレスに余裕がある。

## 3. Windows VM を準備する

既存 VM が使える場合は設定の確認のみ行う。
新規に作る場合:

1. 検索バーの **[仮想マシン] → [作成] → [Azure 仮想マシン]** を開く。[^12]
2. **[基本]** で下表の内容を入力する。

   | 設定欄 | 入力・選択する値 |
   | --- | --- |
   | サブスクリプション / RG | `＜subscription＞` / `＜rg＞` |
   | 仮想マシン名 / リージョン | `＜vm＞` / `＜region＞` |
   | 可用性 / イメージ / サイズ | 組織で承認済みの Windows Server イメージと PoC 用サイズ。可用性は既存ポリシーに従う。 |
   | 管理者アカウント | 承認済みの方式で設定。パスワードを画面共有で開示しない。 |
   | パブリック受信ポート | **[なし]**。インターネット向け RDP 3389 は開けない。 |

3. **[ディスク]** は既定値と暗号化方針を確認。
   **[ネットワーク]** では `＜vnet＞` / `＜vm-subnet＞`、**パブリック IP = [なし]** を選ぶ。
   Bastion または承認済み VPN / ExpressRoute 経由の RDP を利用する。
   Bastion が無い環境でポータル接続のためだけに公開 RDP を作成しない。[^12][^13]
4. **[管理] / [監視] / [拡張機能]** は既存の運用ポリシーに従い、同じログを重複して収集する設定を追加しない。
   **[確認および作成] → [作成]**。
5. **VM → [概要]** で状態と完全なリソース ID、**[ID (Identity)]** でマネージド ID の有無を確認する。
   AMA 導入時のシステム割り当て ID の扱いは既存のユーザー割り当て ID と競合しないか確認する。[^1]

**完了確認:** VM が起動し、承認済みの接続方法から Windows に入れる。
VM に公開 IP と公開 RDP が無い。

## 4. Log Analytics ワークスペースを用意する

1. 検索バー **[Log Analytics ワークスペース] → [作成] → [基本]** を開く。
   既存を利用する場合は **[概要]** で名前、リージョン、サブスクリプション、RG を確認して手順 3 へ。[^14]
2. サブスクリプション `＜subscription＞`、RG `＜rg＞`、名前 `＜law＞`、リージョン `＜region＞` を入力し **[確認および作成] → [作成]**。
3. **ワークスペース → [概要]** から完全なリソース ID と保持・料金の方針を確認する。
   **[ネットワークの分離 (Network Isolation)]** で「公開ネットワークからのデータ取り込み」「公開ネットワークからのクエリ」それぞれの現行値を確認する。
   **この時点では変更しない**。
   既存の診断設定 / 他の DCR / NSP があれば影響確認する。[^4][^5]

**完了確認:** ワークスペースのリージョンは DCR の予定リージョンと同じ。
公開アクセス設定の変更前値が確認され、変更が承認されている。

## 5. AMA の構成取得用 DCE を用意する

1. **[モニター] → [データ収集エンドポイント (Data collection endpoints)]** を開く。
   VM と同一リージョンで流用可能な DCE があれば既存 DCE の名前・既存関連付けを確認し、新規作成は省略。[^2][^3]
2. 新規の場合 **[作成] → [基本]** でサブスクリプション `＜subscription＞`、RG `＜rg＞`、DCE 名 `＜dce＞`、リージョン `＜region＞` を入力して **[確認および作成] → [作成]**。
3. **DCE → [概要]** で構成アクセスの FQDN（`...handler.control.monitor.azure.com` など、**表示された値**）とリソース ID を確認する。
   画面にあるインジェスト FQDN と混同しない。

**完了確認:** DCE が成功状態で VM と同リージョン。
VM との関連付けは手順 8 で行う。[^2][^3]

## 6. AMPLS にワークスペースと DCE を登録する

AMPLS は Private Endpoint から接続する Azure Monitor リソースの範囲を定義するもので、ログの保存先や中継先ではない。[^4][^5]
たとえば今回、`＜law＞` と `＜dce＞` を登録すると、VM の AMA は PE 経由で DCE から構成を取得し、収集したイベントを `＜law＞` に送る。
閉域閲覧端末も PE 経由で `＜law＞` のログをクエリする。[^2][^4]
`Query = Private Only` なら、この PE に接続するネットワークから AMPLS 未登録の別ワークスペースはクエリできない。
`Ingestion = Private Only` も原則スコープ外への送信を制限するが、Log Analytics のリソース固有取り込みには適用されない。
今回のワークスペースへの公開接続を止めるため、公開取り込み・公開クエリは別途無効化する（手順 13）。[^5]

**注意:** AMPLS は共有 DNS / VNet の他のワークスペースへのクエリにも影響し得る。
既存 AMPLS を再利用する場合は追加対象とアクセス モードを変更審査する。
既存スコープのモードを検証のために `Open` へ緩めない。[^4][^5]

1. **[モニター] → [プライベート リンク スコープ (Private Link Scopes)]** を開き、共有 DNS から接続済みの既存 AMPLS を確認する。
   流用する場合は **[アクセス モード]** のクエリ / インジェストの実効値を確認する。
2. 新規作成が承認済みの場合のみ **[作成] → [基本]** でサブスクリプション `＜subscription＞`、RG `＜rg＞`、名前 `＜ampls＞`、**クエリ アクセス モード = Private Only / インジェスト アクセス モード = Private Only** として **[確認と作成] → [作成]**。[^4][^5]
3. **対象 AMPLS → [Azure Monitor リソース] → [追加]** を押し、リソースの一覧で **Log Analytics ワークスペース `＜law＞`** を選択して **[適用]**。
   同じ画面をもう一度開き、**データ収集エンドポイント `＜dce＞`** を選んで **[適用]**。
   DCR や VM をこの一覧へ追加するわけではない。[^2][^4]
4. 一覧に **ワークスペースと DCE が双方存在**することを確認。
   対象外のリソースを追加していないか確認する。

**完了確認:** AMPLS のクエリとインジェストがともに `Private Only` で、ワークスペースおよび DCE がリンクされていることを確認する。
ワークスペースは取り込みとクエリ、DCE は AMA の構成取得の経路に必要。[^2][^5]

## 7. Private Endpoint と Private DNS を構成する

### 7.1 PE の作成

1. **AMPLS → [プライベート エンドポイント接続] → [プライベート エンドポイント]**、または **[プライベート エンドポイント] → [作成]** を開く。[^4]
2. **[基本]** でサブスクリプション `＜subscription＞`、RG `＜rg＞`、PE 名 `＜pe＞`、リージョンに **VNet と同一リージョン**を指定。
   ネットワーク インターフェイス名は組織の命名規則に従う。
3. **[リソース]** で接続方法「ディレクトリ内の Azure リソースに接続」、リソースの種類 `Microsoft.Insights/privateLinkScopes`、対象 `＜ampls＞`、ターゲット サブリソース **`azuremonitor`** を選択する。
   別サブスクリプションの場合は承認済みのリソース ID 方式を使う。[^4]
4. **[仮想ネットワーク]** に `＜vnet＞` / `＜pe-subnet＞` を選ぶ。
   IP は標準の動的割当を使い、静的 IP が必要な場合のみネットワーク担当の割当値を指定する。
   サブネットのネットワーク ポリシー / NSG / UDR を無断で変更しない。
5. **[DNS]** で新規・独立 DNS 構成なら **[プライベート DNS ゾーンとの統合] = [はい]**。
   既存の共有ゾーンや社内 DNS がある場合は、ゾーンと VNet リンクの担当者が既存設定との競合を審査し、承認された統合先を選ぶ。
   **空の A レコードを先に作らない。**
   **[確認および作成] → [作成]**。[^4]
6. **AMPLS → [プライベート エンドポイント接続]** で PE 接続が **承認済み (Approved)** か確認。
   保留なら承認者に依頼し、未承認のまま後続へ進まない。

### 7.2 DNS の照合

1. **[プライベート エンドポイント] → `＜pe＞` → [DNS の構成]** で FQDN とプライベート IP を確認する。
   **[プライベート DNS ゾーン]** からゾーンの **[レコード セット]** と **[仮想ネットワーク リンク]** を確認する。
2. Azure Monitor の代表的なゾーンは `privatelink.monitor.azure.com`、`privatelink.oms.opinsights.azure.com`、`privatelink.ods.opinsights.azure.com`、`privatelink.agentsvc.azure-automation.net`、`privatelink.blob.core.windows.net`。
   **固定の A レコードを推測で作らず、PE が表示する実際の FQDN と IP を正本として照合**する。
   DCE の構成 FQDN、ワークスペースの収集 FQDN、共有クエリ API を重点確認。[^4]
3. VM / 閲覧端末が社内 DNS を使う場合は、Azure DNS への適切な条件付き転送と VNet から PE への経路を担当者に確認する。
   閲覧端末の DNS だけではなく、**閲覧端末が PE に TCP 443 で到達**できなければプライベート クエリは失敗する。[^4]

**完了確認:** PE は Approved。
ゾーンと VNet リンクが存在し、DCE、ワークスペース、クエリ API の名前を VM / 閲覧端末それぞれから正しく解決できる構成になっている。

## 8. DCE を VM に関連付け、DCR を作成して AMA を導入する

### 8.1 構成取得のための VM → DCE

1. **[モニター] → [データ収集エンドポイント] → `＜dce＞` → [リソース]** を開き、VM の既存 DCE 関連付けを確認する。
   VM は構成取得用 DCE を一つだけ関連付けられる。
   別 DCE を追加すると既存の関連付けを置き換えるため、共有 VM であれば事前承認を必ず取得。[^2]
2. **[追加]** で `＜vm＞` を選択し **[適用]**。
   DCR 作成ウィザードの **[リソース]** タブに **[データ収集エンドポイントの有効化]** が表示される場合も、同じ `＜dce＞` が VM に関連付くよう照合する。
   DCR の **[基本]** の DCE 欄はデータ ソース用のインジェスト指定であり、Windows イベント → Log Analytics の構成取得用関連付けとは別。[^1][^3]

### 8.2 VM ゲストの Windows イベントを収集する DCR

1. **[モニター] → [データ収集ルール (Data Collection Rules)] → [作成]**。
   portal 上部に旧画面へのリンクがある場合は**既定の作成画面**を使う。[^1]
2. **[基本]** の設定を入力:

   | 画面の設定欄 | 本 PoC の設定 |
   | --- | --- |
   | ルール名 / サブスクリプション / RG | `＜dcr＞` / `＜subscription＞` / `＜rg＞` |
   | リージョン | `＜region＞`（送信先の Log Analytics と同じ） |
   | テレメトリの種類 | VM の **Windows イベント / パフォーマンス カウンター**を扱えるゲスト ログ用の種類。[選択のヘルプ] で使用可能なデータ ソースと **Azure Monitor ログ**宛先を確認する。 |
   | データ収集エンドポイント（基本欄に表示された場合） | 本 PoC の Windows イベント → Log Analytics 取り込みには指定不要。既定の新画面の指示とヘルプを確認し、手順 8.1 の VM 関連付けと混同しない。 |
   | マネージド ID（DCR 自身） | 本データ ソースが要求する場合だけ設定。VM の ID と同一の項目ではない。 |

3. **[リソース] → [リソースの追加]** で `＜vm＞` を選択。
   ID 選択欄が表示されたら VM の既存 ID 運用に合わせる。
   **[データ収集エンドポイントの有効化]** 欄がある場合は有効にし、当該 VM に `＜dce＞` を指定して、手順 8.1 と一致させる。
   DCR 関連付けで AMA が未導入なら portal が導入する。[^1]
4. **[収集して配信 (Collect and deliver)] → [データ ソースの追加] → [データ ソース]** で **[Windows イベント ログ]** を選ぶ。
   **[基本]** でログ `Application`、レベル `Error` のみ選択。
   `System`、`Security`、`Information` は選択しない。
   **[宛先]** で種類 **[Azure Monitor ログ]**、ワークスペース `＜law＞` を選択しデータ ソースを保存する。
   収集される先は `Event` テーブル。[^1][^6]
5. **任意: CPU グラフも検証する場合のみ**、もう一度 **[データ ソースの追加] → [パフォーマンス カウンター]** を選ぶ。
   カスタムで Windows カウンター `\Processor(_Total)\% Processor Time`、サンプル間隔 60 秒、宛先 **[Azure Monitor ログ] → `＜law＞`** を指定。
   `Perf` テーブルを確認するため、**[Azure Monitor メトリック (プレビュー)]** を宛先の代用にしない。
   不要ならデータ ソース自体を作らない。[^7]
6. **[確認および作成] → [作成]**。
   エラーがあれば必ず解消し、成功として先へ進まない。
   **DCR → [リソース]** で VM が表示され、**DCR → [概要] / [データ ソース]** で Windows イベントと送信先を確認。
   **DCE → [リソース]** に VM が表示されるか、**VM → [拡張機能とアプリケーション]** に `AzureMonitorWindowsAgent` が導入されたかを別々に照合する。[^1][^2]

**完了確認:** VM→DCR、VM→DCE、DCR→`＜law＞`、AMA 拡張機能の 4 点が揃う。
DCR 作成後、データ取り込みには数分（資料上は最大 5 分程度の目安）かかる場合がある。[^1]

## 9. DNS / 疎通と実イベントの取り込みを検証する

### 9.1 VM と閲覧端末を別々に確認

1. **VM 上の Windows PowerShell**（承認済み管理接続）と**閉域の閲覧端末**を用意する。
   PE の **[DNS の構成]** に表示された実際の通常 FQDN を `<fqdn>` として、両端末で次を実行:

   ```powershell
   nslookup <fqdn>
   Test-NetConnection -ComputerName <fqdn> -Port 443
   ```

2. VM では DCE **構成取得**、ワークスペース **収集**の FQDN、閲覧端末ではクエリ API の FQDN を対象にする。
   各 FQDN が想定するプライベート IP に解決され、対象ホストへの `TcpTestSucceeded` が `True` であることを確認する。
   **DNS が解決するだけでは実際の通信経路を証明できない。**[^4]
3. ブラウザーを使用する閲覧端末では **[ワークスペース → ログ]** でクエリ後、開発者ツール / ネットワーク追跡で要求先 API とリモート IP を確認する。
   ブラウザー独自 DNS、DNS キャッシュ、ローカル ネットワーク アクセス許可が OS の `nslookup` と異なる結果を招く場合がある。
   ポータルが開くことだけをプライベート クエリの証拠にしない。[^4]

### 9.2 Windows イベントを 1 件生成

VM の**管理者 Windows PowerShell 5.1**で以下を実行。
ソースが他ログに既存の場合は上書きせず担当者と別名を決定し、後述のクエリの `Source` も同じ名前に変更する。[^9]

```powershell
$source = 'AzureVmDcrPoc'
if (-not [System.Diagnostics.EventLog]::SourceExists($source)) {
    New-EventLog -LogName Application -Source $source
}
Write-EventLog -LogName Application -Source $source -EventId 9001 -EntryType Error -Message 'Azure Monitor DCR PoC test event'
```

**VM → [接続]** の承認済み手段で Windows に入り、**イベント ビューアー → [Windows ログ] → [Application]** を開く。
ソース `AzureVmDcrPoc`、レベル `エラー`、イベント ID `9001` が存在することを確認する。
ローカルに無ければ DCR の設定より先に VM 上の実行結果を調べる。

### 9.3 Logs の実測

1. 閉域の閲覧端末から **[Log Analytics ワークスペース] → `＜law＞` → [ログ (Logs)]** を開き、スコープが対象ワークスペースであることを確認。
   時間範囲は初め **過去 1 時間**（初回設定が遅れた場合は過去 24 時間）にする。
   サンプル クエリの画面が出たら空のクエリ タブへ進む。
2. `<VM_RESOURCE_ID>` を VM の完全な ID へ置き換えて次をそれぞれ貼り付け、**[実行]** する（KQL では引用符を残す）。
   `Heartbeat` は AMA と Azure Monitor の通信を示すが、DCR のイベント収集成功は別途 `Event` で確認する。[^1][^10]

   ```kusto
   Heartbeat
   | where TimeGenerated > ago(30m)
   | where _ResourceId =~ "<VM_RESOURCE_ID>"
   | summarize LastSeen=max(TimeGenerated), Samples=count() by Computer
   ```

   ```kusto
   Event
   | where TimeGenerated > ago(1h)
   | where _ResourceId =~ "<VM_RESOURCE_ID>"
   | where EventLog == "Application" and Source == "AzureVmDcrPoc" and EventID == 9001
   | project TimeGenerated, Computer, EventLevelName, EventID, Source, Message
   | order by TimeGenerated desc
   ```

3. CPU を任意設定した場合のみ、同画面で `Perf` を確認:

   ```kusto
   Perf
   | where TimeGenerated > ago(1h)
   | where _ResourceId =~ "<VM_RESOURCE_ID>"
   | where ObjectName == "Processor" and CounterName == "% Processor Time" and InstanceName == "_Total"
   | summarize AvgCpu=avg(CounterValue) by bin(TimeGenerated, 5m)
   | order by TimeGenerated asc
   ```

   環境固有のカウンター名で 0 件なら、まず `Perf | where TimeGenerated > ago(1h) | summarize count() by ObjectName, CounterName, InstanceName` を実行し実際の列値で修正する。[^7][^10]

**完了確認:** `Heartbeat` の直近 `LastSeen`、`Event` の Event ID 9001 と VM の `_ResourceId` を確認する。
`Perf` を設定しない場合は確認不要。

## 10. Workbook と Azure ダッシュボードを作る

1. **[モニター] → [ブック (Workbooks)] → [新規]** または空テンプレートを開く。
   **[編集] → [追加] → [クエリの追加]** を押す。
   データ ソース **[ログ] / [Azure Monitor ログ]**、対象リソース `＜law＞` を選択し、手順 9 の Heartbeat KQL を入力。
   **[クエリの実行]**、視覚化 **[グリッド]** を選んで **[編集完了]**。[^11]
2. 同じブックにもう一度 **[追加] → [クエリの追加]**。
   データ ソース / リソースを同じワークスペースにし、次の KQL でイベントの時系列を作る。
   **[視覚化] = [時間グラフ]**（画面で利用できる時系列の名称）を選び **[クエリの実行] → [編集完了]**。
   1 件のイベントだけでは点が 1 個なのは正常。[^11]

   ```kusto
   Event
   | where TimeGenerated > ago(24h)
   | where _ResourceId =~ "<VM_RESOURCE_ID>"
   | where EventLog == "Application" and Source == "AzureVmDcrPoc" and EventID == 9001
   | summarize Events=count() by bin(TimeGenerated, 1h)
   | order by TimeGenerated asc
   ```

3. 任意 `Perf` を収集した場合は手順 9 の CPU クエリを第三のパネルへ追加。
   不要なら追加しない。
   画面上部の **[保存]** でタイトル `＜workbook＞`、サブスクリプション、RG、場所を選んで保存し、閉域端末から開き直す。
   **Workbook の閲覧権限と参照先のログ閲覧権限は別々に必要。**[^11][^15]
4. 保存した Workbook を **[編集] → [ピン留め]** モードにし、イベント推移パネル上の **[ピン留め]** を選んで既存 / 新規の Azure ダッシュボード `＜dashboard＞` に追加する。
   **ポータル左メニュー [ダッシュボード]** または検索バー **[ダッシュボード]** から対象を開き、タイルがクエリを表示することを確認する。
   必要なら Heartbeat パネルもピン留めする。[^15]
5. Workbook を後から編集してもピン留め済みタイルの**構成**は自動で更新されない。
   変更したパネルを反映させるときは古いタイルを外して再ピン留めする。
   クエリが参照するデータの更新と、保存されたパネル構成の更新は区別する。[^15]

**完了確認:** Workbook とダッシュボードが閉域閲覧端末で開き、Heartbeat の表とイベント推移が表示できる。
グラフの 0 件はまず手順 9 の KQL と対象スコープを確認する。

## 11. Action Group を作成し、通知経路を単独テストする

1. **[モニター] → [アラート] → [アクション グループ] → [作成]**。
   **[基本]** でサブスクリプション `＜subscription＞`、RG `＜rg＞`、処理リージョン（組織方針）、Action Group 名 `＜ag＞`、表示名を入力。[^8]
2. **[通知]** で **通知の種類 = [電子メール]** を選び、PoC 用の承認済み受信先を入力。
   通知名を設定し **[OK]**。
   共通アラート スキーマは受信側が必要な場合に有効化。
   自動アクション（Webhook、Logic Apps 等）は追加しない。
   **[確認および作成] → [作成]**。[^8]
3. 通知先に確認・認証メールが届いたら受信者に必要な確認操作を依頼。
   **アクション グループ → `＜ag＞` → [テスト]** を実行し、宛先への配信を確認する。
   これは**通知経路だけ**のテストであり、ログ アラートの発火・VM の収集・利用者端末の閉域クエリを証明しない。[^8]

**完了確認:** Action Group のテスト通知を受信できる。
メール不達はアラート作成前に通知先を修正する。

## 12. ログ検索アラートを設定し、実イベントで発火させる

1. **[モニター] → [アラート] → [+ 作成] → [アラート ルール]**。
   **[スコープ] → [リソースの選択]** で `＜law＞` の Log Analytics ワークスペースだけを選択して確定。[^16]
2. **[条件] → [シグナル名] → [カスタム ログ検索 (Custom log search)]**。
   クエリエディターに以下を貼り、`<VM_RESOURCE_ID>` を置換して **[実行] → [アラートの編集を続行する]**。
   イベントが残っている場合はプレビューに行が出るため、作成直後の発火試験前に古いイベントの時間窓を確認する。[^16]

   ```kusto
   Event
   | where TimeGenerated > ago(15m)
   | where _ResourceId =~ "<VM_RESOURCE_ID>"
   | where EventLog == "Application" and Source == "AzureVmDcrPoc" and EventID == 9001
   ```

3. **[条件] → [測定]** は **[テーブル行]**、**[アラート ロジック]** は **静的しきい値 / より大きい / 0**、評価の頻度 **5 分**、集計の粒度（評価対象期間）**15 分**。
   時間窓の初期値が 5 分の場合は明示的に 15 分へ変更。
   1 VM だけなのでディメンションによる分割はしない。[^16]
4. **[アクション] → [アクション グループの追加]** で `＜ag＞` を選択。
   **[詳細]** でアラート名 `＜alert＞`、重大度の例 `Sev 3`、ルール有効化 = オン、説明は「PoC 用 Application / Event ID 9001」とする。
   **[確認および作成] → [作成]**。
   アラートは評価回数に応じた料金を生じ得るため実施後に無効化を予定する。[^16]
5. **古いテスト イベントが 15 分窓の外に出たことを KQL で確認**した後、手順 9.2 の `Write-EventLog` をもう一度実行する。
   `Event` クエリの新規行、**[モニター] → [アラート] → [アラート インスタンス]** の `Fired`、Action Group の通知メールを順に照合する。
   メールが来なくても `Fired` の有無を先に調べる。[^16]

**完了確認:** 新規イベント、`Event` の新規行、`Fired` インスタンス、承認先への通知の四つを同じテストで確認できる。

## 13. 公開アクセスの無効化

### 13.1 「プライベート」の 3 種の設定を区別する

| 設定箇所 | 収集への効果 | クエリへの効果 |
| --- | --- | --- |
| **AMPLS → [アクセス モード]** の Ingestion / Query | `Open` ではスコープ外の公開経路も使える場合がある。`Private Only` でスコープ外を原則遮断。ただし **Log Analytics のリソース固有取り込みエンドポイントには、このモードが適用されない**。 | `Private Only` は同じ PE / DNS のネットワークからスコープ外監視リソースのクエリを遮断し得る。 |
| **ワークスペース → [ネットワークの分離]** の公開取り込み / 公開クエリ | 取り込みを Disabled にすると対象ワークスペースの公開接続を拒否する。 | クエリを Disabled にすると対象ワークスペースの公開接続を拒否する。 |
| **Private DNS と実際の TCP 経路** | AMA の DCE 構成取得 / ワークスペース取り込みの FQDN が PE に向くことを確認する。 | 閲覧端末の共有クエリ API が PE に向くことを確認する。 |

**AMPLS の `Private Only` だけで Log Analytics への任意ワークスペース向け公開取り込みを止めたとは言えない。**
他ワークスペース宛ての送信抑止が要件ならネットワーク FW で公開エンドポイント向け送信の遮断も別途設計する。[^5]

### 13.2 公開アクセスの切替手順（共有基盤の場合は承認必須）

1. 作業前に、手順 9～12 が Private Endpoint 経由で動くこと、同じ DNS を使う他のワークスペース・DCE・閲覧端末が把握されていること、切り戻し方法と承認者が明確であることを確認する。
   **一つでも不明なら変更しない。**[^4][^5]
2. **AMPLS → [アクセス モード]** で、**クエリ = Private Only**、**インジェスト = Private Only** であることを確認する。
   **[除外 (Exclusions)]** に PE 個別のアクセス モードがあれば、その PE の実効設定を確認する。[^4]
3. **[Log Analytics ワークスペース] → `＜law＞` → [ネットワークの分離 (Network Isolation)]** を開き、**公開ネットワークからの取り込み = いいえ / 無効**、**公開ネットワークからのクエリ = いいえ / 無効** にそれぞれ変更して **[保存]**。
   画面名称が違う場合は `publicNetworkAccessForIngestion` と `publicNetworkAccessForQuery` が両方 `Disabled` に相当することを確認する。
   既存 DCE にも公開アクセス設定がある場合は構成取得への影響を確認し、承認済みの場合のみ無効化する。[^4][^5]
4. VM からの `Heartbeat` / 新規 `Event` / 任意 `Perf` の取り込み、閉域閲覧端末からの Logs / Workbook / ダッシュボード表示、**新しい**イベントを起点としたアラート発火を**もう一度**試す。
   `Fired` が継続中なら再発火試験に使わず、アラート状態がリセットされていることを確認してから試す。[^16]
5. **許可された非接続の試験端末**で対象ワークスペースの公開 DNS / 公開経路を確認し、認証後に同一クエリが拒否されることを確認する。
   非接続端末が実は VPN や企業プロキシから PE に到達する場合は陰性試験にならない。
   ポータルのログイン成功だけでは判定しない。
   非接続 VM を追加してまで公開取り込みの拒否試験を行わない場合、公開側取り込み拒否は未試験として扱う。[^4][^5]
6. 閉域側で取り込みまたはクエリが失敗した場合、影響を拡大する変更を追加せず原因を確認し、担当者の承認を得て切替前の設定値に戻す。
   PE / Private DNS を共有環境から即時削除しない。[^4]

**合格:** 対象ワークスペースの公開取り込み・クエリが Disabled で、閉域側の実通信が PE 経由で成功し、権限と経路を確認した非接続端末からのクエリが拒否される。

## 14. 切り分けと終了処理

| 症状 | 最初に確認する場所 / 順序 |
| --- | --- |
| PE 承認待ち / DNS が公開 IP | **AMPLS → [プライベート エンドポイント接続]** の承認 → **PE → [DNS の構成]** → Private DNS zone / VNet リンク → 社内 DNS の条件付き転送。 |
| DCE 構成が取れず `Heartbeat` が無い | **DCE → [リソース]** の VM 関連付け、AMA 拡張機能、VM の ID、DCE の構成 FQDN の VM 側名前解決 / 443。AMA の詳細な状態は VM 内ログの承認済み調査手順へ。 |
| Heartbeat はあるが `Event` が 0 件 | Windows **イベント ビューアー → Application** で 9001 を確認 → **DCR → [リソース]** の VM → **[データ ソース]** の Application/Error → ワークスペース宛先 → KQL の `_ResourceId` / `Source` / 時間範囲。 |
| `Event` はあるがポータル Logs / Workbook が失敗 | 閲覧端末の PE への経路 / ブラウザー DNS / ローカル ネットワーク アクセス許可 → ワークスペースの公開クエリ設定 → Log Analytics 閲覧権限。Action Group のメール受信はクエリ経路の証拠にならない。 |
| `Event` はあるがアラートが発火しない | アラートの有効化、スコープ、KQL の 15 分窓、5 分頻度、古いアラートの状態を確認。`Fired` がありメールだけ無ければ Action Group / 宛先を確認。[^16] |

終了時は **[モニター] → [アラート] → [アラート ルール] → `＜alert＞`** で PoC 用ルールを**無効化**して不要な通知・評価料金を防ぐ。
既存資産を削除しない。
新規作成した VM、ワークスペース、DCE、AMPLS、PE、DNS、Workbook、ダッシュボード、Action Group、DCR の削除・保持はオーナーに確認してから実施する。
イベント ソースは別用途で使われていないかを確認するまで削除しない。

## 参考資料（Microsoft Learn）

[^1]: Azure Monitor を使用して仮想マシンからゲスト ログ データを収集する, https://learn.microsoft.com/ja-jp/azure/azure-monitor/vm/data-collection
[^2]: 仮想マシンと Kubernetes クラスターのプライベート リンクを有効にする, https://learn.microsoft.com/ja-jp/azure/azure-monitor/fundamentals/private-link-vm-kubernetes
[^3]: Azure Monitor のデータ収集エンドポイント, https://learn.microsoft.com/ja-jp/azure/azure-monitor/data-collection/data-collection-endpoint-overview
[^4]: Azure Monitor のプライベート リンクを構成する, https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/private-link-configure
[^5]: Azure Monitor のプライベート リンク構成を設計する / Azure Private Link を使用してネットワークを Azure Monitor に接続する, https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/private-link-design ; https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/private-link-security
[^6]: Azure Monitor を使用して仮想マシンから Windows イベントを収集する, https://learn.microsoft.com/ja-jp/azure/azure-monitor/vm/data-collection-windows-events
[^7]: Azure Monitor を使用して仮想マシンからパフォーマンス カウンターを収集する, https://learn.microsoft.com/ja-jp/azure/azure-monitor/vm/data-collection-performance
[^8]: Azure Monitor アクション グループ, https://learn.microsoft.com/ja-jp/azure/azure-monitor/alerts/action-groups
[^9]: New-EventLog / Write-EventLog (Windows PowerShell 5.1), https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-eventlog?view=powershell-5.1 ; https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/write-eventlog?view=powershell-5.1
[^10]: Heartbeat / Event / Perf テーブル リファレンス, https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/heartbeat ; https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/event ; https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/perf
[^11]: Azure Workbook を作成または編集する, https://learn.microsoft.com/ja-jp/azure/azure-monitor/visualize/workbooks-create-workbook
[^12]: Azure portal で Windows VM を作成する, https://learn.microsoft.com/ja-jp/azure/virtual-machines/windows/quick-create-portal
[^13]: Azure portal で Azure Bastion をデプロイする, https://learn.microsoft.com/ja-jp/azure/bastion/tutorial-create-host-portal
[^14]: Log Analytics ワークスペースを作成する, https://learn.microsoft.com/ja-jp/azure/azure-monitor/logs/quick-create-workspace
[^15]: Azure Monitor Workbooks を管理する（ピン留めと共有）, https://learn.microsoft.com/ja-jp/azure/azure-monitor/visualize/workbooks-manage
[^16]: ログ検索アラート ルールを作成または編集する, https://learn.microsoft.com/ja-jp/azure/azure-monitor/alerts/alerts-create-log-alert-rule
