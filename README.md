# dameng-db

Dameng DM8 database skill for Codex and other agents that support [skills.sh](https://skills.sh/).

It helps agents work on Dameng database installation, migration, SQL/DDL rewriting, Java/Spring/MyBatis adaptation, post-migration triage, performance tuning, operations, monitoring, and cluster architecture.

## Install

Recommended non-interactive install:

```bash
npx --yes skills add hexinmiao96/dameng-db -g --skill dameng-db
```

List the skill before installing:

```bash
npx skills add hexinmiao96/dameng-db --list
```

### Fallback: Git clone fails

`npx skills add` clones the GitHub repository. If your network can open GitHub in a browser but Git clone fails with errors like `Recv failure: Connection reset by peer` or `Failed to connect to github.com port 443`, install from the GitHub zip archive:

```bash
tmp_dir="$(mktemp -d)"
trap 'rm -rf "$tmp_dir"' EXIT
dest="${CODEX_HOME:-$HOME/.codex}/skills/dameng-db"

curl -L -o "$tmp_dir/dameng-db.zip" \
  https://codeload.github.com/hexinmiao96/dameng-db/zip/refs/heads/main
unzip -q "$tmp_dir/dameng-db.zip" -d "$tmp_dir"

if [ -e "$dest" ]; then
  echo "Destination already exists: $dest"
  echo "Remove or back it up before reinstalling."
  exit 1
fi

mkdir -p "$(dirname "$dest")"
cp -R "$tmp_dir/dameng-db-main" "$dest"
```

Verify after installation:

```bash
npx skills ls -g --json | jq '.[] | select(.name == "dameng-db")'
```

Or verify directly:

```bash
test -f "${CODEX_HOME:-$HOME/.codex}/skills/dameng-db/SKILL.md"
sed -n '1,12p' "${CODEX_HOME:-$HOME/.codex}/skills/dameng-db/SKILL.md"
```

## When to Use

- Migrate Oracle, MySQL, PostgreSQL, SQL Server, or DB2 projects to Dameng DM8.
- Rewrite SQL, DDL, MyBatis XML, Java datasource config, and ORM settings for Dameng.
- Triage migration issues such as reserved keywords, table aliases, `GROUP BY`, `IDENTITY`, generated IDs, `reference` columns, and MyBatis `<if>` conditions.
- Check installation, tablespace/user management, backup, monitoring, performance tuning, and cluster deployment plans.
- Separate reusable Dameng compatibility risks from pure project-specific bugs.

## Example Prompts

```text
Use $dameng-db to review these MyBatis XML files for Dameng compatibility risks.
```

```text
Use $dameng-db to triage this post-migration issue list and classify which items are reusable Dameng migration rules.
```

```text
Use $dameng-db to build a Dameng DM8 migration checklist for a Spring Boot + MyBatis project.
```

```text
Use $dameng-db to check this SQL for reserved words, table aliases, GROUP BY issues, and IDENTITY insert risks.
```

## Coverage

- Installation and initialization
- Tablespaces, users, schemas, and permissions
- Migration from Oracle/MySQL/PostgreSQL/SQL Server/DB2
- SQL and DDL compatibility rewriting
- Java, Spring, MyBatis, MyBatis-Plus, Hibernate, and other framework adaptation
- PL/SQL, system packages, and programming APIs
- Post-migration triage and reusable risk extraction
- Performance, monitoring, operations, and cluster architecture

## Notes

- The skill is designed for reusable Dameng database work. It intentionally excludes pure project bugs that are unrelated to Dameng compatibility.
- For production or source databases, use read-only access unless explicit write permission is granted.
- Public examples use placeholders instead of real passwords or private network addresses.
