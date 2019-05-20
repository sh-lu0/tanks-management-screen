## VagrantでCentOS7環境を作成
https://qiita.com/sudachi808/items/3614fd90f9025973de4b

ログイン
`vagrant ssh`

## 共有ファイル
- Vagrantfileを編集
ホスト側のVagrantfileと同じ階層のdataファイルと、
ゲスト側の/vagrant_dataが共有される
