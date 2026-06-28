# Linux Lesson 03 — Text Processing: Pipes, grep, sed & awk

| | |
|---|---|
| **Track** | Linux |
| **Lesson** | 03 of 15 |
| **Topic** | The Unix philosophy of composable text tools: redirection, pipes, grep, sed, awk, sort, uniq, cut |
| **Estimated time** | ~2 hours |
| **Prerequisites** | Lessons 01–02 |

### Learning objectives
1. Use streams, redirection (`>`, `>>`, `<`, `2>`, `2>&1`) and pipes (`|`) fluently.
2. Search text with `grep` and basic regular expressions.
3. Transform text with `sed`; extract/compute with `awk`.
4. Combine `sort`, `uniq`, `cut`, `tr`, `wc`, `head`, `tail` into pipelines.
5. Build real log-analysis one-liners — the daily bread of ops/data work.

---

## 1. Why this matters
The Unix philosophy — *small tools that do one thing well, composed via pipes* — is the most leveraged skill on the command line. Parsing logs, extracting fields from CSVs, summarizing data, building quick reports: a one-line pipeline often replaces a 30-line script. For DevOps and data engineering this is a daily, high-value skill and a frequent live-coding interview task.

---

## 2. Theory

### 2.1 Streams and redirection
Every process has three standard streams:
- **stdin (0)** — input, **stdout (1)** — normal output, **stderr (2)** — errors.

```bash
cmd > out.txt        # redirect stdout to a file (overwrite)
cmd >> out.txt       # append
cmd < in.txt         # feed file as stdin
cmd 2> err.txt       # redirect stderr
cmd > out.txt 2>&1   # stdout AND stderr to same file (order matters!)
cmd &> all.txt       # bash shorthand for both
cmd 2>/dev/null      # discard errors (/dev/null = the void)
```

### 2.2 Pipes — the core idea
`|` connects one command's stdout to the next command's stdin, forming a **pipeline**. Data flows left to right; each stage transforms the stream.
```bash
cat access.log | grep "500" | wc -l        # how many 500 errors
```
(Often you can skip `cat`: `grep "500" access.log | wc -l`.)

### 2.3 Regular expressions (regex) — the pattern language
Used by `grep`, `sed`, `awk` (different from shell globbing!). Core syntax:
- `.` any char · `*` zero+ of previous · `+` one+ (ERE) · `?` optional (ERE)
- `^` start of line · `$` end of line · `[abc]` set · `[^abc]` negated set · `[0-9]` range
- `\d` is *not* standard in basic grep; use `[0-9]`. `\b` word boundary (GNU).
- **BRE** (basic) vs **ERE** (extended): use `grep -E` (or `egrep`) for `+ ? | ()` without backslashes.

### 2.4 grep — search lines
```bash
grep "ERROR" app.log               # lines containing ERROR
grep -i "error" app.log            # case-insensitive
grep -r "TODO" src/                # recursive through a directory
grep -n "panic" app.log            # show line numbers
grep -c "200" access.log           # count matching lines
grep -v "DEBUG" app.log            # invert: lines NOT matching
grep -E "WARN|ERROR" app.log       # extended regex alternation
grep -o "[0-9]\{3\}" access.log    # print only the matched part
grep -A2 -B2 "Exception" app.log   # 2 lines After/Before context
```

### 2.5 sed — stream editor (transform lines)
```bash
sed 's/foo/bar/' file          # replace FIRST foo per line
sed 's/foo/bar/g' file         # replace ALL (global)
sed 's/foo/bar/gi' file        # global + case-insensitive
sed -n '10,20p' file           # print only lines 10–20 (-n suppresses default print)
sed '/^#/d' config             # delete comment lines
sed -i 's/old/new/g' file      # edit file IN PLACE (careful!)
sed -E 's/([0-9]+)/<\1>/g'     # ERE with capture group backreference \1
```

### 2.6 awk — field-oriented processing & computation
`awk` splits each line into fields (`$1`, `$2`, … `$NF` = last; `$0` = whole line) and runs `pattern { action }`.
```bash
awk '{print $1}' access.log              # first field of each line
awk -F',' '{print $2}' data.csv          # comma-delimited → 2nd column
awk '$3 > 100 {print $1, $3}' data       # filter on a numeric field
awk '{sum += $5} END {print sum}' data   # total a column
awk -F',' 'NR>1 {a[$2]+=$3} END{for(k in a) print k, a[k]}' sales.csv  # group-by sum
awk 'NR==1 || /ERROR/' app.log           # header line OR error lines
```
`NR` = current record (line) number; `NF` = number of fields; `BEGIN{}`/`END{}` run before/after.

### 2.7 The supporting cast
```bash
sort file              # sort lines; sort -n numeric; sort -r reverse; sort -k2 by field 2
uniq                   # collapse ADJACENT duplicates (sort first!); uniq -c counts
cut -d',' -f1,3 data   # extract columns 1 and 3 (delimiter ,)
tr 'a-z' 'A-Z'         # translate/transform characters; tr -d to delete
wc -l / -w / -c        # count lines / words / bytes
head -n / tail -n      # first/last N lines; tail -f to follow
xargs                  # turn stdin into arguments for another command
```

The canonical "top N" idiom:
```bash
... | sort | uniq -c | sort -rn | head    # frequency count, most common first
```

---

## 3. Official documentation quotes

> "grep searches for PATTERNS in each FILE. PATTERNS is one or more patterns separated by newline characters, and grep prints each line that matches a pattern."
> — *grep(1) man page* (GNU grep)

> "sed is a stream editor. A stream editor is used to perform basic text transformations on an input stream (a file or input from a pipeline)."
> — *GNU sed manual*, [gnu.org/software/sed](https://www.gnu.org/software/sed/manual/sed.html)

> "The awk utility shall execute programs written in the awk programming language, which is specialized for textual data manipulation. An awk program is a sequence of patterns and corresponding actions."
> — *POSIX / The Open Group Base Specifications*, [awk](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/awk.html)

> "The Unix philosophy: Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams, because that is a universal interface."
> — Doug McIlroy, as summarized in the Unix tradition.

---

## 4. Real-world examples

### 4.1 Top 10 IPs hitting a web server
```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
```

### 4.2 Count HTTP status codes
```bash
awk '{print $9}' access.log | sort | uniq -c | sort -rn
# e.g.  14233 200 / 512 404 / 87 500
```

### 4.3 Extract and summarize errors with timestamps
```bash
grep -i "error" app.log | sed -E 's/^([0-9-]+T[0-9:]+).*/\1/' | sort | uniq -c
```

### 4.4 Sum a CSV column for a group
```bash
# sales.csv: date,region,amount
awk -F',' 'NR>1 {rev[$2]+=$3} END {for (r in rev) printf "%-10s %.2f\n", r, rev[r]}' sales.csv
```

### 4.5 Find unique error messages
```bash
grep "ERROR" app.log | cut -d']' -f2- | sort -u
```

---

## 6. How This Is Used In Production
- **Startups:** before there's a fancy logging stack, engineers debug production by SSHing in and running `grep`/`awk` pipelines over `/var/log`. It's the fastest way to answer "how many 500s in the last hour and from which endpoint."
- **Enterprises:** even with centralized logging (ELK/Splunk/Loki), engineers drop to `grep`/`awk` on a host for fast, ad-hoc triage; these tools also power glue scripts in ETL and CI. Splunk/SPL and `awk` solve the same shape of problem.
- **Common architectures:** log files → `tail`/`grep` for live triage; batch jobs use `awk`/`cut`/`sort` for lightweight ETL and report generation; pipelines feed `xargs` to parallelize work across files.
- **Scaling:** these tools stream (constant memory, like Python generators in Python Lesson 03), so they handle multi-GB logs a GUI editor can't open. For truly huge data you graduate to distributed tools, but the *mental model* (map/filter/reduce over a stream) is identical.
- **Monitoring/Logging:** alerting scripts often boil down to `grep -c PATTERN | threshold check`; log shippers pre-filter with grep-like rules; `awk` computes quick metrics (p95 from response-time columns) during incidents.
- **Security:** parsing auth logs for failed logins (`grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn`) is a classic intrusion-detection one-liner; fail2ban automates exactly this pattern.
- **Real examples:** SREs at **Google**, **Netflix**, **Cloudflare** and virtually everywhere use these pipelines for incident triage; **Cloudflare** and CDN operators routinely `awk` over enormous access logs. The "sort | uniq -c | sort -rn" frequency idiom is industry folklore for a reason.

---

## 7. Best practices
- Build pipelines **incrementally** — add one stage at a time and inspect output (pipe to `head`).
- Quote regex/patterns to protect them from the shell: `grep "^ERROR"`.
- Prefer `grep -E` (ERE) for readable alternation/quantifiers.
- Use `awk` when you need *fields* or *arithmetic*; `grep` to *find*; `sed` to *edit*. Don't force one tool to do all three.
- Be careful with `sed -i` (in-place) — test without `-i` first, or use `sed -i.bak` to keep a backup.
- Remember `uniq` only collapses **adjacent** duplicates — `sort` first.

## 8. Common mistakes & gotchas
- `uniq` without `sort` — misses non-adjacent duplicates.
- Confusing glob `*` with regex `.*` — in `grep`, `*` means "zero+ of the previous char."
- Forgetting `-r` for recursive grep, or `-n` for line numbers.
- `sed -i` clobbering a file with a wrong pattern and no backup.
- Word-splitting/quoting bugs: unquoted variables and patterns break on spaces.
- Assuming `\d` works in basic grep — use `[0-9]` or `grep -P` (PCRE, if available).
- Parsing structured formats (JSON, CSV with quoted commas) with awk/cut — use `jq`/a real parser for those.

## 9. Where AI helps (and where it hurts)
- **Helps:** generating an `awk`/`sed`/`grep` one-liner from a plain-English description, explaining a cryptic pipeline someone else wrote, building the regex, and suggesting the right tool for the job.
- **Hurts:** AI regexes can be subtly wrong (greedy matches, escaping for the wrong dialect BRE/ERE/PCRE) — always test on sample data. It may suggest `sed -i` without a backup. For JSON/CSV it may hand you a fragile awk hack where `jq`/`csvkit` is correct. Verify output against a few known lines.

## 10. Learn independently
- [GNU grep](https://www.gnu.org/software/grep/manual/), [GNU sed](https://www.gnu.org/software/sed/manual/sed.html) manuals.
- [The GNU Awk User's Guide (gawk)](https://www.gnu.org/software/gawk/manual/) — surprisingly readable.
- [regex101.com](https://regex101.com/) to build/test patterns; [explainshell.com](https://explainshell.com/) to dissect pipelines.
- *The Linux Command Line* (Shotts), text-processing chapters; *Classic Shell Scripting* (Robbins & Beebe).

## 11. Interview preparation

**Q1. What does the pipe `|` do?**
> It connects the stdout of the left command to the stdin of the right command, letting you compose small tools into a pipeline that transforms a data stream stage by stage.

**Q2. Difference between `>` and `>>` and `2>&1`?**
> `>` redirects stdout to a file, overwriting; `>>` appends. `2>&1` redirects stderr to wherever stdout currently points (so `> file 2>&1` sends both to the file). Order matters because `2>&1` copies the *current* destination of stdout.

**Q3. When use grep vs sed vs awk?**
> `grep` to find/filter lines by pattern; `sed` to do line-oriented text substitution/editing on a stream; `awk` when you need to work with fields/columns or do arithmetic/aggregation. They overlap, but each has a sweet spot.

**Q4. Why must you `sort` before `uniq`?**
> `uniq` only removes *adjacent* duplicate lines. Sorting brings identical lines together so `uniq` (often with `-c` to count) works correctly.

**Q5. Write a one-liner for the top 5 most frequent values in column 1 of a space-delimited file.**
> `awk '{print $1}' file | sort | uniq -c | sort -rn | head -5`

**Q6. How do you sum the 3rd column of a CSV (skipping the header)?**
> `awk -F',' 'NR>1 {s+=$3} END {print s}' file.csv`

**Q7. Glob vs regex `*` — what's the difference?**
> In shell globbing, `*` matches any sequence of characters in filenames. In regex (grep/sed/awk), `*` means "zero or more of the *preceding* element"; "any sequence" is `.*`. They're different languages.

## 12. Homework
> Generate or download a sample web access log / CSV; save work in `linux/solutions/lesson-03/`.

**Easy**
1. Given a log file: (a) count total lines; (b) count lines containing "ERROR" (case-insensitive); (c) show the last 20 lines and any line numbers where "WARN" appears.

**Medium**
2. From a web `access.log`, produce: (a) the top 10 client IPs by request count, (b) a count of each HTTP status code, and (c) all distinct URLs that returned 404. Use pipelines of `grep`/`awk`/`sort`/`uniq`/`cut`. Save each pipeline and its output.

**Hard**
3. Write a single pipeline (or short set) that, from a CSV `sales.csv` (`date,region,product,amount`), outputs total revenue **per region** sorted descending, formatted to 2 decimals — using `awk` associative arrays. Then write an `auth.log` analysis that lists usernames with the most failed SSH login attempts. Document each tool's role in comments.

**Stretch:** Reproduce a "p95 response time" estimate: from a log with a response-time column, use `awk` + `sort -n` to compute the 95th-percentile value, and explain the approach. Compare your awk arithmetic approach with what you'd do in SQL (ties into SQL Lesson 06 window functions).

## 13. Key takeaways
- Streams (stdin/stdout/stderr) + redirection + pipes = composable data processing.
- `grep` finds, `sed` edits, `awk` handles fields and arithmetic — pick the right tool.
- Regex (tools) ≠ globbing (shell); `*` means different things in each.
- `sort | uniq -c | sort -rn | head` is the universal frequency-count idiom.
- These tools stream in constant memory — they scale to logs too big to open in an editor — and are the fastest path to answers during incidents.
