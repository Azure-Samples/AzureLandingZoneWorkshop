# 워크샵 에이전트 작업 지침

## 리포 규칙

시크릿을 커밋하지 않습니다. 실제 키와 리소스 식별자는 환경 변수와 예시 placeholder로 구분합니다. 미커밋 변경과 저자의 실습 내용을 보존합니다.

코드와 문서는 모두 `LICENSE.md`의 MIT 조건을 따릅니다. Azure-Samples 표준 저작권 표기를 보존하고, 별도 승인 없이 라이선스나 저작권 문구를 변경하지 않습니다.

`SECURITY.md`, `CODE_OF_CONDUCT.md`, `SUPPORT.md`의 신고·문의 창구 TODO는 유지보수자가 실제 운영할 경로를 확인한 뒤 채웁니다. Microsoft의 연락처, 다른 워크샵의 issue tracker나 응답 기한을 이 저장소의 정책으로 임의 지정하지 않습니다.

랩 경로는 `NN-kebab-name`을 사용합니다. 랩 전용 자산은 해당 랩의 `assets/`에 두고 여러 랩이 참조하는 자산만 `shared-assets/`에 둡니다.

## 콘텐츠·노트북 스타일

이 저장소는 Azure Portal이 주 학습 경로인 A형 워크샵입니다. 불필요한 실행 환경이나 노트북을 추가하지 않습니다. 루트 README는 전체 메타데이터를 사용하고, 랩 README는 `title`, `duration_minutes`, `last_updated`를 사용합니다.

소요 시간과 비용은 저자가 허용한 계획용 추정과 실측을 구분합니다. 이 워크샵의 240분은 저자가 제공한 전체 시간이며 랩별 시간은 운영 추정입니다. 비용 추정에는 리전, SKU, 단가 조회일, 사용시간과 제외 항목을 명시합니다. 실행 결과를 추정해 확정하지 않습니다. 공식 문서와의 대조를 실제 Azure Portal 화면이나 E2E 검증으로 표시하지 않습니다.

## 검증 하네스

검사 환경은 Bash 기반 devcontainer 또는 Git Bash입니다. gitleaks 8.19 이상, Python 3.10 이상과 YAML 검사에 필요한 PyYAML을 준비합니다. 설치나 업데이트는 승인받고, 도구가 없어 실행하지 못한 검사는 미실행으로 보고합니다.

```bash
# (1) 시크릿 스캔 — 이력 포함 (gitleaks >= 8.19 신문법)
gitleaks git . --log-opts="--all" || { echo "FAIL: secrets"; exit 1; }

# (2) 개인 리소스 식별자 스캔 — 문서·코드·설정
grep -rEn --include='*.md' --include='*.py' --include='*.ipynb' --include='*.sh' --include='*.bicep' --include='*.json' \
  'https?://[a-z0-9-]+\.(openai\.azure\.com|cognitiveservices\.azure\.com|services\.ai\.azure\.com|azure-api\.net)' . \
  | grep -v '<your\|example\|placeholder' && echo "FAIL: personal endpoints" || echo "PASS"

# (3) 노트북 출력 클리어 확인
python3 - <<'EOF'
import json, glob, sys
bad = [f for f in glob.glob('**/*.ipynb', recursive=True)
       if any(c.get('outputs') or c.get('execution_count') is not None
              for c in json.load(open(f))['cells'] if c['cell_type']=='code')]
print("FAIL: outputs in", bad) if bad else print("PASS: notebooks clean")
sys.exit(1 if bad else 0)
EOF

# (4) 공백·비ASCII 경로 잔존 확인
find . -path ./.git -prune -o -print | grep -P '[ ]|[^\x00-\x7F]' \
  && echo "FAIL: non-standard paths" || echo "PASS"

# (5) frontmatter 스키마 검증
python3 - <<'EOF'
import re, sys, yaml
txt = open('README.md', encoding='utf-8').read()
m = re.match(r'^---\n(.*?)\n---', txt, re.S)
assert m, "FAIL: no frontmatter"
fm = yaml.safe_load(m.group(1))
required = ['type','title','description','level','authors','contacts','duration_minutes',
            'tags','language','execution','status','source','last_updated']
missing = [k for k in required if k not in fm]
if 'validated_on' not in fm: missing.append('validated_on(키 필수, 값은 공란 허용)')
print("FAIL: missing", missing) if missing else print("PASS: frontmatter")
sys.exit(1 if missing else 0)
EOF

# (6) 깨진 상대 링크·이미지 참조 검사 (fenced/inline 코드 제외, <img src> 포함)
python3 - <<'EOF'
import re, glob, os, sys, urllib.parse
bad = []
for f in glob.glob('**/*.md', recursive=True):
    t = open(f, encoding='utf-8').read()
    t = re.sub(r'```.*?```', '', t, flags=re.S)
    t = re.sub(r'`[^`\n]*`', '', t)
    base = os.path.dirname(f)
    links = re.findall(r'\]\(([^)#\s]+?)(?:#[^)]*)?\)', t)
    links += re.findall(r'<img[^>]+src=["\']([^"\']+)["\']', t)
    for link in links:
        if link.startswith(('http', 'mailto:', 'data:')): continue
        p = os.path.normpath(os.path.join(base, urllib.parse.unquote(link.strip())))
        if not os.path.exists(p): bad.append(f"{f} -> {link}")
print("FAIL:\n" + "\n".join(bad)) if bad else print("PASS: links")
sys.exit(1 if bad else 0)
EOF

# (7) 랩 frontmatter 검증
python3 - <<'EOF'
import re, glob, os, sys, yaml
bad = []
for f in glob.glob('**/README.md', recursive=True):
    d = os.path.basename(os.path.dirname(f))
    if not re.match(r'^(\d{2}-|lab\d)', d): continue
    m = re.match(r'^---\n(.*?)\n---', open(f, encoding='utf-8').read(), re.S)
    if not m: bad.append(f + ' -> frontmatter 없음'); continue
    try: fm = yaml.safe_load(m.group(1)) or {}
    except yaml.YAMLError: bad.append(f + ' -> YAML 파싱 오류(따옴표 누락 등)'); continue
    miss = [k for k in ('title','duration_minutes','last_updated') if k not in fm]
    if miss: bad.append(f + ' -> 키 누락: ' + ','.join(miss))
print("FAIL:\n" + "\n".join(bad)) if bad else print("PASS: lab frontmatter")
sys.exit(1 if bad else 0)
EOF

# (8) 이미지 alt-text 검사 (빈 alt 금지 — markdown ![]와 HTML <img>)
python3 - <<'EOF'
import re, glob, sys
bad = []
for f in glob.glob('**/*.md', recursive=True):
    t = re.sub(r'```.*?```', '', open(f, encoding='utf-8').read(), flags=re.S)
    if re.search(r'!\[\s*\]\(', t): bad.append(f + ' -> 빈 alt (markdown)')
    for img in re.findall(r'<img[^>]*>', t):
        m = re.search(r'alt=["\']([^"\']*)["\']', img)
        if not m or not m.group(1).strip(): bad.append(f + ' -> 빈 alt (<img>)')
print("FAIL:\n" + "\n".join(bad)) if bad else print("PASS: alt-text")
sys.exit(1 if bad else 0)
EOF

# (9) 랩 전용 자산 교차 참조 검사
python3 - <<'EOF'
import re, glob, os, sys, urllib.parse
bad = []
for f in glob.glob('**/*.md', recursive=True):
    t = re.sub(r'```.*?```', '', open(f, encoding='utf-8').read(), flags=re.S)
    base = os.path.dirname(f).replace(os.sep, '/')
    links = re.findall(r'\]\(([^)#\s]+?)\)', t) + re.findall(r'<img[^>]+src=["\']([^"\']+)["\']', t)
    for l in links:
        if l.startswith(('http','mailto:','data:')) or '/assets/' not in l or 'shared-assets' in l: continue
        p = os.path.normpath(os.path.join(base, urllib.parse.unquote(l))).replace(os.sep, '/')
        owner = p.split('/assets/')[0]
        if not (base + '/').startswith(owner + '/'):
            bad.append(f + ' -> ' + l + ' (shared-assets 이동 대상)')
print("FAIL:\n" + "\n".join(bad)) if bad else print("PASS: assets")
sys.exit(1 if bad else 0)
EOF
```

## 백로그·DO NOT

- `validated_on`은 사람이 완주한 날짜만 기록합니다.
- 삭제, 분할, 라이선스 전환, 이력 변경은 승인 없이 수행하지 않습니다.
- 검사를 생략하거나 완화하지 않고, 미실행을 통과로 표시하지 않습니다.
- 클라우드 배포·삭제, 유료 실행, 원격 push와 공개 전환은 별도 승인 없이 수행하지 않습니다.
- TODO가 남아 있는 골격을 검증된 실습이나 공개 준비 완료로 표시하지 않습니다.
