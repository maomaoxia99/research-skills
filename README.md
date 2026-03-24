# research-skills

A Claude Cowork Plugin with useful research and utility skills.

## Skills Included

| Skill | Description | Trigger Keywords |
|-------|-------------|-----------------|
| product-research | 产品调研十维分析框架 | 产品调研、竞品分析、market research |
| shenzhen-weather | 深圳天气查询助手 | 深圳天气、SZ weather、深圳温度 |
| okx-meme-antirug | Meme 防 Rug 扫链定投 | meme 防rug、扫链买币、meme antirug |

## Install

### Install all skills

```bash
npx skills add maomaoxia99/research-skills
```

### Install a single skill

```bash
# 产品调研十维分析框架
npx skills add maomaoxia99/research-skills/product-research

# 深圳天气查询助手
npx skills add maomaoxia99/research-skills/shenzhen-weather

# Meme 防 Rug 扫链定投
npx skills add maomaoxia99/research-skills/okx-meme-antirug
```

## Project Structure

```
research-skills/
├── .claude-plugin/
│   └── plugin.json                        # Root plugin (all skills)
├── product-research/
│   ├── .claude-plugin/
│   │   └── plugin.json                    # Standalone plugin
│   └── skills/
│       └── product-research/
│           └── SKILL.md
├── shenzhen-weather/
│   ├── .claude-plugin/
│   │   └── plugin.json                    # Standalone plugin
│   └── skills/
│       └── shenzhen-weather/
│           └── SKILL.md
├── okx-meme-antirug/
│   ├── .claude-plugin/
│   │   └── plugin.json                    # Standalone plugin
│   └── skills/
│       └── okx-meme-antirug/
│           └── SKILL.md
├── skills/
│   ├── product-research/
│   │   └── SKILL.md
│   ├── shenzhen-weather/
│   │   └── SKILL.md
│   └── okx-meme-antirug/
│       └── SKILL.md
├── package.json
└── README.md
```

## How to Add a New Skill

1. Create `skills/<skill-name>/SKILL.md` (for root plugin)
2. Create `<skill-name>/.claude-plugin/plugin.json` (standalone plugin metadata)
3. Create `<skill-name>/skills/<skill-name>/SKILL.md` (standalone plugin skill file)
4. Update this README with install commands
5. Bump `package.json` version, push, and create a Release

## License

MIT
