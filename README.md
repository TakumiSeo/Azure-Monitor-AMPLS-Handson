# Azure VM ログ監視 / AMPLS ハンズオン

Windows VM のイベントログを Azure Monitor Agent（AMA）とデータ収集ルール（DCR）で Log Analytics に収集するハンズオンです。

閉域でのログ閲覧とアラート通知を検証します。

![Azure VM ログ監視の構成図](assets/azure-vm-dcr-ampls-private-only.png)

- VM の AMA は DCE から構成を取得します。
- 収集したログは Log Analytics に送信します。
- 閉域閲覧端末は Private Endpoint 経由で Log Analytics をクエリします。
- AMPLS の Query / Ingestion は `Private Only` にします。
- Log Analytics の公開クエリ / 公開取り込みは無効にします。

詳細は [Azure portal 操作手順書](verification/azure-vm-dcr-ampls-portal-runbook.md) を参照してください。

構成図の編集用原本は [draw.io ファイル](assets/azure-vm-dcr-ampls-private-only.drawio) です。
