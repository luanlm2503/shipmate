# shipmate

Agent skill: biến host agent thành **first-mate** — điều phối crew worker agents chạy song song, mỗi con một git worktree + Herdr pane riêng, giám sát tới xong rồi báo cáo gộp.

Xây dựng hoàn toàn trên Herdr CLI native (`herdr worktree`, `herdr workspace`, `herdr agent`) — không cần firstmate repo, WSL hay tmux. Chạy trên Windows / Linux / macOS ở đâu có Herdr.

## Yêu cầu

- Herdr server đang chạy, tương thích (`herdr status`)
- `jq` có trên `PATH` (`jq --version`) — để parse ID từ output JSON của Herdr

## Cài đặt skill

Copy `SKILL.md` vào thư mục skill của agent bạn dùng:

| Agent | Đích |
|-------|------|
| Claude Code | `~/.claude/skills/shipmate/SKILL.md` |
| Codex / OpenCode | `~/.agents/skills/shipmate/SKILL.md` |

Hoặc symlink repo này thẳng vào đó.

## Khi nào dùng

- Có 2+ subtask độc lập (fix, investigate, refactor, audit) chạy song song không đụng file nhau.
- User nói "spin up agents", "run a crew", "delegate to workers" hoặc tương tự.
- 1 task dài, muốn main session rảnh tay trong lúc worker chạy nền.

Không dùng cho 1 task nhỏ, hoặc khi các subtask overlap file nặng (worktree cách ly git state, không giải quyết merge conflict).

## Workflow (tóm tắt)

1. **Plan** — chẻ request thành unit độc lập: task id (`[a-z][a-z0-9_-]{0,31}`), objective + stop condition cụ thể, chọn kind (`claude` / `opencode` / `codex`). Hỏi user worktree đặt ở đâu.
2. **Dispatch** — mỗi task: `herdr worktree create` → `herdr agent start <id> --kind <kind> --pane <pane> -- <trust-flag>` → `herdr agent prompt <id> "<objective>" --wait --until idle --until blocked`.
3. **Supervise** — `herdr agent wait <id> --until idle --until blocked --until done`. Nếu `blocked`, `agent read` xem dialog rồi `agent send-keys` hoặc prompt sửa.
4. **Collect & verify** — `agent read` + verify độc lập trong worktree (`git status`, test/build/lint). Không tin summary của crewmate.
5. **Report** — báo cáo gộp: task id, kind, branch, kết quả verify, đề xuất merge / follow-up / xem tay.
6. **Clean up** — chỉ sau khi user confirm: `herdr worktree remove --workspace <id>` (+ kiểm tra `herdr workspace list` để đóng workspace lẻ).

Chi tiết đầy đủ, prompt skeleton, trust flag từng kind và các pitfall đã verify (`agent_prompt_stalled` race, PATH shadowing, stray workspace) xem trong [`SKILL.md`](SKILL.md).

## Pitfall đã biết (đã verify bằng test)

- `herdr agent prompt --wait` có thể trả `agent_prompt_stalled` / `timeout` dù crewmate đã làm đúng — coi là "unknown, go check": chạy `herdr agent get <id>` đọc `agent_status` thật, `idle`/`done` thì đi tiếp bình thường.
- Nếu `--kind` có nhiều bản cài global trên PATH mà 1 bản hỏng, `agent start` có thể timeout — sanity-check kind 1 lần trên pane/worktree dùng 1 lần trước khi dispatch hàng loạt.
- `cd` vào worktree trước `herdr worktree create` có thể tạo workspace lẻ trỏ cùng path — dọn bằng `herdr workspace list` + `workspace close`.

## License

MIT.
