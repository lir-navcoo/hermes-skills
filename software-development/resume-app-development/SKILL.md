---
name: resume-app-development
description: 李睿简历应用的开发工作流，包含GitHub推送、数据结构、渲染模式
---

# Resume App Development Workflow

## Context
github.com/lir-navcoo/resume，GitHub Pages (branch: `gh-pages`)
技术栈：React + TypeScript + Tailwind CSS + Vite + shadcn/ui

## GitHub Push Workflow (Critical)
**Always verify build locally BEFORE pushing to GitHub.**

```bash
cd /tmp/resume-build && cp /tmp/resume_app.tsx src/App.tsx && npm run build
```

If build fails, fix TypeScript errors locally. Push only after clean build.

**Push via GitHub API** (gh cli git-credential doesn't work):
```bash
SHA=$(gh api "repos/lir-navcoo/resume/contents/src/App.tsx?ref=main" --jq '.sha')
CONTENT=$(base64 -i /tmp/resume_app.tsx)
gh api --method PUT "repos/lir-navcoo/resume/contents/src/App.tsx" \
  -f message="commit message" -f content="$CONTENT" -f sha="$SHA"
```

## Local Dev Setup
```bash
cd /tmp && git clone --depth=1 https://TOKEN@github.com/lir-navcoo/resume.git resume-build
cd resume-build && npm install
# Edit src/App.tsx, then:
cp /tmp/resume_app.tsx src/App.tsx && npm run build
```

## Data Structure: Multi-Role Companies (Bilingual)

Use `companies` array for companies with multiple roles, `experience` array for single-role companies. All text fields have bilingual variants (`*En` suffix).

```typescript
const resumeData = {
  companies: [
    {
      name: '公司名',
      nameEn: 'Company Name (English)',
      tags: ['美团'],  // optional, rendered on company header
      roles: [
        { period: '2024.10 – 至今', periodEn: 'Oct 2024 – Present', role: '职位', roleEn: 'Role EN', description: '...', descriptionEn: '...', achievement: '', achievementEn: '' },
      ],
    },
  ],
  experience: [
    { company: '其他公司', companyEn: 'Other Company', period: '...', periodEn: '...', role: '...', roleEn: '...', description: '...', descriptionEn: '...', tags: [] },
  ],
}
```

**Key rules:**
- **DO NOT use `as const`** on the resumeData object — it causes TypeScript to infer `never` type for arrays after filtering, breaking skill rendering
- Use `(exp as any).achievement` for optional fields
- `companies` and `experience` are sibling arrays in resumeData (not nested)

## Bilingual Rendering Pattern

Use a helper `L(zh, en)` for all user-facing text:

```typescript
const L = (zh: string, en: string) => lang === 'en' ? en : zh
const t = i18n(lang)  // t for static UI labels

// In JSX:
<span>{L('技能清单', 'Skills')}</span>
<p>{L('优秀成果', 'Achievement')}: {L(role.achievement, role.achievementEn)}</p>

// For skills category (categoryEn may be undefined):
<p>{L(skill.category, skill.categoryEn || skill.category)}</p>
```

**CRITICAL: For numbered list items, use `itemsEn` array — NOT `L(text, '')`**.  
Chinese uses `1、` numbering, English uses `1.` — regex splitting fails silently in English mode because the separator patterns don't match.

**Data structure for roles with numbered lists — always use `itemsEn`:**
```typescript
roles: [
  {
    period: '2020.09 – 2021.05', periodEn: 'Sep 2020 – May 2021',
    role: '客户成功经理', roleEn: 'Customer Success Manager',
    description: '1、负责执行公司整体客户策略。2、负责存量客户续费指标...', descriptionEn: '...',
    itemsEn: ['Executed overall company customer strategy', 'Managed renewal targets for existing customers...', '...'],  // <-- always add this
    achievement: '...', achievementEn: '...',
  }
]
```

**JSX rendering — always prefer `itemsEn`:**
```tsx
{role.description.includes('。') && role.description.match(/^[0-9]、/) ? (
  <ol className="mt-2 space-y-1 list-decimal list-inside text-sm">
    {role.description.split(/(?=[0-9]、)/).filter(s => s.trim()).map((item, idx) => (
      <li key={idx}>
        {L(item.replace(/^[0-9}、]+/, '').trim(), (role as any).itemsEn?.[idx] || item.replace(/^[0-9}、]+/, '').trim())}
      </li>
    ))}
  </ol>
) : (
  <p>{L(role.description, role.descriptionEn)}</p>
)}
```

## Print Tip Pattern

Download button with hover tooltip showing print tip and hotkey:
```tsx
<div className="relative group">
  <button onClick={handlePrint} ... title={L('下载 PDF / 快捷键 ' + t.hotKey, 'Download PDF / Hotkey ' + t.hotKey)}>
    {/* icon */}
  </button>
  <div className="absolute right-0 top-full mt-2 w-64 px-3 py-2.5 rounded-xl border shadow-lg opacity-0 invisible group-hover:opacity-100 group-hover:visible transition-all duration-200 z-50"
    style={{ background: cardBg, borderColor: cardBorder, color: textSecondary, fontSize: '11px' }}>
    <p className="font-medium mb-1" style={{ color: textPrimary }}>{L('📌 ' + t.printTipTitle, '📌 ' + t.printTipTitle)}</p>
    <p>{L('为增强效果，请打印时勾选"打印背景"', t.printTip)}</p>
    <p className="mt-1" style={{ color: textMuted }}>{L('快捷键: ' + t.hotKey, 'Hotkey: ' + t.hotKey)}</p>
  </div>
</div>
```

## JSX Rendering Pattern (Unified, Bilingual)

All experience entries render in a single `space-y-4` container (no separate sections for companies vs experience).

```tsx
<section>
  <h2 className="text-lg font-semibold mb-3 flex items-center gap-2" style={{ color: textPrimary }}>
    <span className="w-1 h-5 rounded-full inline-block" style={{ background: sectionAccent }} />
    {L('工作经历', 'Experience')}
  </h2>
  <div className="space-y-4">

    {/* Multi-role companies */}
    {resumeData.companies.map((company, ci) => (
      <div key={'co-' + ci} className="rounded-xl border" style={{ background: cardBg, borderColor: cardBorder }}>
        <div className="px-5 py-3 border-b flex items-center" style={{ borderColor: cardBorder }}>
          <p className="font-medium" style={{ color: textPrimary }}>{L(company.name, company.nameEn)}</p>
          {(company.tags || []).map(tag => (
            <span key={tag} className={`ml-2 text-[10px] px-1.5 py-0.5 rounded font-medium ${tagMeituan}`}>
              {L(tag, tag === '美团' ? 'Meituan' : tag)}
            </span>
          ))}
        </div>
        <div className="divide-y" style={{ borderColor: cardBorder }}>
          {company.roles.map((role, ri) => (
            <div key={'role-' + ri} className="p-5">
              <div className="flex items-start justify-between gap-3 flex-wrap">
                <p className="font-medium text-sm" style={{ color: textSecondary }}>{L(role.role, role.roleEn)}</p>
                <span className="text-xs whitespace-nowrap mt-0.5" style={{ color: textMuted }}>{L(role.period, role.periodEn)}</span>
              </div>
              {role.description.includes('。') && role.description.match(/^[0-9]、/) ? (
                <ol className="mt-2 space-y-1 list-decimal list-inside text-sm" style={{ color: textSecondary }}>
                  {role.description.split(/(?=[0-9]、)/).filter(s => s.trim()).map((item, idx) => (
                    <li key={idx}>{L(item.replace(/^[0-9}、]+/, '').trim(), '')}</li>
                  ))}
                </ol>
              ) : (
                <p className="text-sm mt-1 leading-relaxed" style={{ color: textSecondary }}>{L(role.description, role.descriptionEn)}</p>
              )}
              {role.achievement && (
                <div className="mt-3 px-3 py-2 rounded-lg border" style={{ background: isDark ? 'rgba(59,130,246,0.1)' : 'rgba(59,130,246,0.06)', borderColor: isDark ? 'rgba(59,130,246,0.25)' : 'rgba(59,130,246,0.2)' }}>
                  <p className="text-xs font-medium" style={{ color: isDark ? '#93c5fd' : '#3b82f6' }}>
                    {L('优秀成果', 'Achievement')}: {L(role.achievement, role.achievementEn)}
                  </p>
                </div>
              )}
            </div>
          ))}
        </div>
      </div>
    ))}

    {/* Single-role entries - same card style */}
    {resumeData.experience.map((exp, i) => (
      <div key={'exp-' + i} className="rounded-xl border" style={{ background: cardBg, borderColor: cardBorder }}>
        <div className="px-5 py-3 border-b flex items-center" style={{ borderColor: cardBorder }}>
          <p className="font-medium" style={{ color: textPrimary }}>{L(exp.company, exp.companyEn)}</p>
        </div>
        <div className="p-5">
          <div className="flex items-start justify-between gap-3 flex-wrap">
            <p className="font-medium text-sm" style={{ color: textSecondary }}>{L(exp.role, exp.roleEn)}</p>
            <span className="text-xs whitespace-nowrap mt-0.5" style={{ color: textMuted }}>{L(exp.period, exp.periodEn)}</span>
          </div>
          <p className="text-sm mt-1 leading-relaxed" style={{ color: textSecondary }}>{L(exp.description, exp.descriptionEn)}</p>
        </div>
      </div>
    ))}
  </div>
</section>
```

**Note:** Both single-role and multi-role cards use the same visual style (company header + role body). Single-role entries still render inside a company-header div for visual consistency.

## TypeScript Pitfalls

- **DO NOT use `as const` on resumeData** — causes TypeScript to infer `never` type for filtered/transformed arrays, breaking skill rendering. Use plain object literal.
- **Password variable**: When doing global find-replace, ensure `CORRECT_PASSWORD` is not accidentally overwritten (e.g. with `'***'`). Always verify it reads `const CORRECT_PASSWORD='navcoo'` before pushing.
- **Optional `achievement`**: Use `(exp as any).achievement` since TypeScript doesn't know about optional fields
- **Numbered descriptions**: Check `exp.description.match(/^[0-9]、/)` to render as `<ol>` vs plain text
- **Duplicate entries in arrays**: When adding bilingual fields, be careful not to accidentally create duplicate array entries (e.g. two identical skill categories with only English/Chinese split). Remove all duplicates before pushing.

## Section Order
About → Skills → Experience (Companies + Single-role merged) → Original Projects → GitHub Projects → Education → Footer

## Avatar
Use DingTalk CDN URL directly (no local file needed):
```tsx
<img src="https://static.dingtalk.com/media/lADPDhmOzuuAPhjNAt3NAtw_732_733.jpg" alt={resumeData.name} className="w-full h-full object-cover" />
```

## Common File Operations

```bash
# Upload file to repo
gh api --method PUT "repos/lir-navcoo/resume/contents/path/to/file" \
  -f message="msg" -f content="$(base64 -i file.jpg | tr -d '\n')"

# Get file SHA before update
gh api "repos/lir-navcoo/resume/contents/path?ref=main" --jq '.sha'
```
