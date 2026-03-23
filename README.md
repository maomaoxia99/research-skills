# research-skills

A Claude Cowork Plugin with useful research and utility skills.

## Skills Included

| Skill | Description | Trigger Keywords |
|-------|-------------|-----------------|
| product-research | 产品调研十维分析框架 | 产品调研、竞品分析、market research |
| shenzhen-weather | 深圳天气查询助手 | 深圳天气、SZ weather、深圳温度 |

## Install

```bash
npm install research-skills
```

Or install directly from GitHub:

```bash
npm install maomaoxia99/research-skills
```

## Project Structure

```
research-skills/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata
├── skills/
│   ├── product-research/
│   │   └── SKILL.md         # 产品调研十维分析框架
│   └── shenzhen-weather/
│       └── SKILL.md         # 深圳天气查询助手
├── package.json
└── README.md
```

## How to Add a New Skill

1. Create a new folder under `skills/` with your skill name
2. Add a `SKILL.md` file with frontmatter (name, description) and instructions
3. Update `package.json` version
4. Publish to npm: `npm publish`

## License

MIT
