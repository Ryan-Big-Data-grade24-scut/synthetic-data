# 虚构账单样例：生成方法

**先模拟交易，再导出流水。** 不随机拼一行“某天花了多少钱”，而是先回答：谁因为什么，用哪个账户付给谁？钱从哪里来，最后到哪里去？

## 核心流程

1. **先定主体、账户和期初状态。** 定义家庭成员、雇主、商户和摊主、平台、银行、贷款方、投资产品方等。账户明确归属；期初现金、历史负债、持仓和可退税额事先登记。期初不算当期收入，余额不足也不凭空补钱。
2. **让日历和随机规则触发业务，不直接写流水。** 固定周期触发发薪、月供和账单；随机规则决定何时买什么、向谁买、用什么账户付。业务层检查合同、余额、授信和原单。结算要么整体生效，要么不动钱；未付或取消可以留状态，不能伪造成成功交易。
3. **记一次资金流，再生成各渠道的凭证。** 每笔结算记录现金从哪到哪；负债、份额和估值另外记。例如：支付宝绑卡买了 ¥120，发卡方垫付、商户收款、持卡人新增 ¥120 债务。支付宝和信用卡各有一行，但只算一次消费；还卡只清偿债务；退 ¥35 必须冲回原卡，而不是转进支付宝余额。
4. **把真实用途、平台标签和可见证据分开。** 答案按业务用途划分收入、义务、刚性、弹性；日、月、季、半年、年都保留毛支出、到账退款和净支出。导出只展示各格式实际能提供的字段，有交易对方就直接写在流水里。平台标签故意制造关键词误判、把个体摊主压进通用类别；错分率只算可比较的消费主行。答案知道哪两行是同一笔，不代表受测方看得出来：原字段直证或可见字段能唯一匹配时才给这项关系评分，多解和缺证不算错。
5. **做三种独立检查。** 手算关键场景，检查退款、卡账和申赎规则；从期初重放全年现金和非现金状态，检查是否守恒、余额和份额是否对得上；读回原始 CSV/XLSX，逐行检查金额、字段及重复次数。最后用同一代码版本、依赖、种子和区间在新目录重跑，比对原始导出及答案文件的 SHA-256。SQLite 不要求字节一致，单独用真实导入器验证支持的文件。

## 复现与取用

这是独立的虚构账单样例仓库。`20261126-v3-through-20261125/` 含 **2026-01-01 至 2026-11-25（含）** 的 19 份原始导出（6,478 行）、标准答案和核验记录，不含真实账单或运行数据库。11 月、第四季度、下半年与年度汇总均为截至 11 月 25 日的累计值，不代表完整期间。

生成器在 [`ai-finance-assistant`](https://github.com/Ryan-Big-Data-grade24-scut/ai-finance-assistant/tree/20f665ca38ed180c1e946617628bbaf4f8e49c80/scripts/synth)。复现时把两个仓库克隆到同一父目录，再从 `ai-finance-assistant` 根目录用项目虚拟环境运行（Git Bash）。命令行的 `--months` 只能按月结束，这里用引擎指定截止日：

```bash
.venv/Scripts/python.exe -X utf8 - <<'PY'
from datetime import date
from pathlib import Path
from scripts.synth.engine import Ledger
from scripts.synth.writers import write_all

out = Path('../synthetic-data/repro-through-20261125')
dataset = Ledger(20261126, date(2026, 1, 1), date(2026, 11, 25)).run()
write_all(dataset, out)
PY
```

输出目录必须不存在或为空；重跑换新目录。若还要复现隔离导入，用 `python -m scripts.synth.backend_ledger --dataset <输出目录> --out <新的隔离库目录>`；用 `python -m scripts.synth.backend_check --dataset <输出目录>` 可以让**真实导入器 + 真实账本引擎**复算一遍账，再和答案层逐类对账。导入器现已覆盖 18 份原始流水（支付宝、微信、小荷包、借记卡、信用卡、电商订单、基金、证券、理财收益）；资产快照是某一时点的持仓表，不是流水，不入库。复算结果见 `verification/backend_ledger_totals.csv`：收入、义务支出、刚性支出、弹性支出四类金额与答案层**一分不差**。

- **盲测输入**：从 [正式样例](20261126-v3-through-20261125/) 只取支付宝、微信、支付宝小荷包、银行卡、美团、淘宝天猫、京东、投资目录。
- **出题方留存**：[`ground_truth/ANSWER.md`](20261126-v3-through-20261125/ground_truth/ANSWER.md) 看简要答案；`ground_truth/` 查逐笔和周期汇总；`reconciliation.csv` 查原始行覆盖；`verification/` 查手算、重放、逐文件日期范围和哈希证据。`verification/full_year/` 是生成器的固定目录名，重放只覆盖本样例的实际区间。隔离库不入 Git。不要把整个仓库发给受测方；正式盲测另用未公开的种子。
- **改生成规则**：先看 [交易模型](https://github.com/Ryan-Big-Data-grade24-scut/ai-finance-assistant/blob/main/scripts/synth/MODEL.md) 和 [答案与评分口径](https://github.com/Ryan-Big-Data-grade24-scut/ai-finance-assistant/blob/main/scripts/synth/GROUND_TRUTH.md)。先改主体、业务条件和结算规则，不在导出文件里修数字。生成只使用虚构数据，不读取真实账本，也不覆盖旧样例。
