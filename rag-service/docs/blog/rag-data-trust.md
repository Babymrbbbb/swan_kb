# 从「语料与索引不一致」到数据可信：RAG 知识库的可靠性方法

> 技术标签：FastAPI / RAG / 数据治理 / 评测闭环 / 可靠性
> 适用读者：正在做 RAG/知识库问答，并关心「数据可信」而不是「demo 能跑」的工程同学。

---

## 一、典型故障：答案为什么会长期错？

一个知识问答 RAG 服务，被问到「覆盖哪些枚举 / 共多少网点」时长期返回一份与源文件不符的旧答案。

问题在于：**语料里的分布是人工编造的、从未 grounding 到权威数据；且改正源文件后索引未重建**。更隐蔽的是：即使后来把源文件改正了，**线上检索依旧返回旧答案**——因为 RAG 的索引是「源文件 → 灌库脚本」生成的静态产物，改 `.md` 不会自动改动向量库里的 `document`/`chunk` 表。**源改了，索引没动**，这是 RAG 项目最常见也最致命的坑。

---

## 二、根因拆解：不是一层问题，是四层叠加

| 层 | 问题 | 本质 |
|---|---|---|
| 内容层 | corpus 里的数字/枚举是**凭空编的**，从未接地到权威数据 | 数据来源不可信 |
| 索引层 | 改源后**未重建索引**，线上查的仍是旧库 | 源与索引脱节 |
| 测试层 | pytest 把语料重灌进**临时库**，只证明「源文件对了」，持久化库从未被测试覆盖 | 测试绿 ≠ 线上对 |
| 修复层 | 上次纠正只改了部分文件，漏掉聚合页/概述里的同一事实 | 修复不完整 |

结论：**RAG 的「答案质量」首先不是模型问题，而是数据与工程问题。**

---

## 三、解法一：单一真相源（Single Source of Truth）

把业务数据从「散落在多个 md 里」收敛为**唯一真相源**：
- 门店/网点数据 → `data/store_facts.json`（含总数、枚举分布、不存在的枚举清单）
- 其它业务事实（日期、门槛、规格…）→ `data/business_facts.json`

真相源是**唯一更新入口**。配套生成器 `scripts/generate_stores_md.py` 从真相源**重建**视图文件，杜绝手写漂移。

---

## 四、解法二：启动自愈合，让索引永远跟得上源

索引 stale 的根治方案：**应用启动时比对语料哈希，漂移即重建**。

```python
# app/seed.py 核心逻辑（简化）
def reseed_if_needed(session):
    current = corpus_hash()          # 对 data/corpus/*.md 求稳定哈希
    stored  = get_meta("corpus_hash")
    doc_count = count_documents()
    if stored == current and doc_count > 0:
        return False                 # 一致，跳过
    rebuild_corpus(session)          # 清旧建新 + 写回最新哈希
    return True
```

`corpus_hash()` 对所有 md 按文件名排序后取 SHA-256，保证确定性；`Meta` 表存上一次哈希，启动时比对。效果：源文件一改，下次启动自动重建索引。

---

## 五、解法三：数据接地测试，把错误卡在 CI

自愈只能保证「索引和源一致」，但**源本身如果就是错的呢**？所以还要有接地测试——校验 corpus 与真相源的一致性：

```python
# tests/test_grounding.py（数据无关，不写死任何数字）
def test_no_fabricated_city():
    text = corpus_text()
    for city in facts["nonexistent_cities"]:
        assert city not in text   # 真相源说「不存在」的枚举，corpus 不得出现

def test_store_total_consistent():
    totals = re.findall(r"(?:拥有|全国)\s*(\d+)\s*家门店", corpus_text())
    assert totals
    assert all(int(t) == facts["total_stores"] for t in totals)

def test_distribution_matches_truth():
    for city, n in facts["distribution"].items():
        assert city.split("（")[0] in corpus_text()
        assert str(n) in corpus_text()
```

关键是**数据无关**：不写死任何具体数字/枚举，而是动态读取真相源。因此数据更新（总数变化、新增枚举）测试自然通过，漏改某个文件则被自动抓出并阻断合并。业务事实同样处理：`tests/test_business_grounding.py` 要求真相源里每条事实的关键词必须出现在 corpus。

---

## 六、解法四：评测闭环，让质量可回归

数据可信之后，还要证明「检索真的准」：
- `data/eval_golden.json`：覆盖核心场景的问题 + 期望命中文档 + 期望事实提示词
- `app/eval.py`：跑「检索命中率」（期望文档是否在 top-k 召回内）与「忠实度」（期望事实是否出现在答案或召回原文）
- 评测结果落库为 QA 审计记录，可复盘 Badcase

离线 Mock 确定性评测可稳定复现（结果见 `docs/eval-report.md`）。评测还能发现**Badcase**：例如某类问题的 top1 召回是概述页而非期望的专题页，虽然 top-k 覆盖了期望文档（命中率仍 1.0），但暴露「top1 精度可提升」——改进方向是给该维度加权。**能发现这种 case，正是评测闭环存在的意义。**

---

## 七、复盘：RAG 数据可信的 checklist

1. [ ] 业务数据是否可追溯到权威源？（还是某次人工编的？）
2. [ ] 改源后是否自动重建索引？（还是靠人记得跑脚本？）
3. [ ] 测试是否覆盖持久化库？（还是只测临时库？）
4. [ ] 修复是否全量？（grep 全仓确认没有漏网文件？）
5. [ ] 是否有数据接地测试卡 CI？（错误数据在合并前就被拦截？）
6. [ ] 是否有 golden-set 评测闭环？（质量可回归，改坏了能发现？）

---

## 八、关键收获

1. RAG 的答案质量，首先取决于数据链路，而不是模型；把「语料→索引→检索→生成」每一环都变成可验证的，比调 prompt 更重要。
2. 单一真相源 + 生成器 + 数据无关测试，是一套可长期演进的数据治理模式。
3. 启动自愈合是消除「源-索引脱节」的可靠手段，成本极低（一次哈希比对）。
4. 评测闭环的价值在 Badcase——指标达标不等于没问题。

---

*本文描述 RAG 知识服务的数据可信方法，技术实现见 hunkon_rag。技术栈：FastAPI + SQLModel + SQLite + pytest。*
