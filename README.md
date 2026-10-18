# مهارات التطوير الجاهزة لبولت — Jev Pro

مستودع يضم مهارات (Agent Skills) يستعملها مساعد التطوير على مشروع **Jev Pro**.
كل مهارة = مجلد يحوي `SKILL.md` (صيغة المهارات القياسية التي يفهمها Bolt).

## المهارات

**أساسيات التطوير**: brainstorming · writing-plans · executing-plans · test-driven-development ·
systematic-debugging · verification-before-completion · requesting-code-review · receiving-code-review ·
refactoring · working-effectively-with-legacy-code · performance-optimization · api-and-interface-design ·
context-engineering · mcp-builder · webapp-testing

**تصميم واجهات**: frontend-design · ui-ux-pro-max (نسخة مخففة أقل من 2MB) · jev-dashboard-design

**خاصة بالمشروع**: jev-pro-context · voice-pipeline-debugging · llm-streaming-client · python-windows-audio

## الاستيراد إلى Bolt

### الطريقة الأولى — دفعة واحدة (الأسهل)
1. حمّل الملف: `bolt-skills.zip` (في جذر هذا المستودع) — زر Download.
2. في Bolt: صفحة Skills → Add skill → **الاستيراد من ملف** → اختر `bolt-skills.zip` → Import.

### الطريقة الثانية — من GitHub
1. في Bolt: صفحة Skills → Add skill → **من جيت هاب**.
2. Repository URL: `https://github.com/YOUSSFE9/bolt-skills`
3. اختر اسم مجلد المهارة → Create. (يتطلب أن يكون المستودع عاماً — كما هو الآن)

> ملاحظة: حد المهارة الواحدة في الاستيراد من GitHub هو 2MB، لذلك `ui-ux-pro-max` مخففة
> (أُزيلت ملفات بيانات ضخمة غير أساسية وبيانات التحقق؛ تبقى الوظائف الأساسية).

## هيكل مستودع المشروع
لكل مهارة مجلد مستقل في الجذر يحوي `SKILL.md` — بهذا يتعرف Bolt عليها تلقائياً عند الاستيراد.
