# AGENTS.md — apps リポの運用ルール

運用ルールの正本は **`CLAUDE.md`**（同内容）。他CLI（Codex等）で起動した場合もそれを読むこと。

要点だけ：

- ここは趣味アプリの実験場。**このリポ配下への書き込みは自由**。
- **ボードは無い**。進行は各アプリの `SPEC.md` / `CHANGELOG.md` に残す。
- 役を分けて動く（mira/nia/rei/sena/sara/advisor）。詳細は `.claude/agents/`。
- 判断3レベル：L1=自走／L2=advisor／**L3=ren だけ**（外部公開・課金・secret・削除・Git破壊的操作・Emmaリポへの書き込み）。
- `../Emma/` は**読み取り専用**。
