# تقرير إصلاح مشكلة التقريب في تطبيق KSA Compliance (ZATCA)

يوثق هذا الملف التعديلات الجذرية التي تمت على تطبيق `ksa_compliance` لحل مشكلة الرفض من هيئة الزكاة (ZATCA) المتعلقة بالعمليات الحسابية للأصناف والخصومات، وتحديداً الخطأ الشائع:
`[BR-KSA-EN16931-07]-Item net price (BT-146) must equal (Item Gross price (BT-148) - Allowance amount (BT-147))`

## 🔍 التفسير التقني للمشكلة (السبب الجذري)
كان النظام يقوم بحساب السعر الصافي للصنف `PriceAmount` داخل كود البايثون باستخدام دالة `round()` القياسية، بينما يتم معالجة باقي القيم (السعر الأساسي، ومبلغ الخصم) داخل ملف الـ `XML` باستخدام دالة `rounded()` الخاصة بـ Frappe. 
* دالة `round()` في بايثون تستخدم "التقريب البنكي" (Banker's Rounding) حيث يتم تقريب النصف (0.5) لأقرب رقم زوجي.
* دالة `rounded()` في واجهة Frappe تقوم بالتقريب بطريقة مختلفة قليلاً في بعض الحالات العشرية الدقيقة.

**النتيجة السابقة (قبل التعديل):**
كانت القيم تخرج من النظام إلى زاتكا بهذا الشكل:
* السعر الأساسي (BaseAmount): `165217.39`
* الخصم (Allowance Amount): `91304.34`
* السعر الصافي المحسوب في البايثون (PriceAmount): `73913.04`

عندما تقوم هيئة زاتكا بالتدقيق رياضياً:
`165217.39 - 91304.34 = 73913.05`
وبما أن الناتج يختلف عن المكتوب (`73913.04`) بفارق هللة واحدة (0.01)، يتم رفض الفاتورة فوراً.

---

## 🛠 التعديلات التي تمت

### 1. تعديل ملف البايثون
**المسار:** `ksa_compliance/output_models/e_invoice_output_model.py`

**الوضع قبل التعديل:**
تمت محاولة حساب السعر الصافي داخل البايثون عن طريق تقريب كل قيمة على حدة:
```python
if item["discount_amount"]:
    rounded_disc = round(abs(item["discount_amount"]), 2)
    rounded_base = round(abs(item["base_amount"]), 2)
    item["net_price_for_xml"] = round(rounded_base - rounded_disc, 2)
```

**الوضع بعد التعديل:**
تم إزالة هذا الكود الخاطئ بالكامل، والاعتماد فقط على إرسال القيم الخام للقالب (Template) ليتم المعالجة هناك بشكل موحد.
```python
# تم إزالة الحسبة من البايثون لمنع تضارب دوال التقريب
# Note: PriceAmount (net_price_for_xml) is computed in the Jinja template using
# rounded(base_amount,2) - rounded(discount_amount,2)
```

---

### 2. تعديل قالب فاتورة زاتكا (XML)
**المسار:** `ksa_compliance/templates/e_invoice.xml`

**الوضع قبل التعديل:**
كان القالب يأخذ `PriceAmount` الجاهز من البايثون، ويقوم بتقريب الخصم والسعر الأساسي بمفردهما:
```xml
<cac:Price>
    <cbc:PriceAmount currencyID="{{ invoice.currency_code }}">{{ item.net_price_for_xml }}</cbc:PriceAmount>
    <cbc:BaseQuantity unitCode="PCE">{{ item.qty }}</cbc:BaseQuantity>
    {% if item.discount_amount %}
    <cac:AllowanceCharge>
        <cbc:ChargeIndicator>false</cbc:ChargeIndicator>
        <cbc:Amount currencyID="{{ invoice.currency_code }}">{{ rounded(item.discount_amount, 2) }}</cbc:Amount>
        <cbc:BaseAmount currencyID="{{ invoice.currency_code }}">{{ rounded(item.base_amount, 2) }}</cbc:BaseAmount>
    </cac:AllowanceCharge>
    {% endif %}
</cac:Price>
```

**الوضع بعد التعديل:**
تم ضبط القالب ليقوم بفرض حقيقة رياضية (Price = Base - Discount) وتمريرها جميعاً عبر نفس دالة التقريب الخاصة بفريمورك فرابي `rounded()` لضمان استحالة حدوث فارق الهللة المزعج:
```xml
<cac:Price>
    {% if item.discount_amount %}
    {%- set r_base = rounded(item.base_amount, 2) -%}
    {%- set r_disc = rounded(item.discount_amount, 2) -%}
    <cbc:PriceAmount currencyID="{{ invoice.currency_code }}">{{ rounded(r_base - r_disc, 2) }}</cbc:PriceAmount>
    <cbc:BaseQuantity unitCode="PCE">{{ item.qty }}</cbc:BaseQuantity>
    <cac:AllowanceCharge>
        <cbc:ChargeIndicator>false</cbc:ChargeIndicator>
        <cbc:Amount currencyID="{{ invoice.currency_code }}">{{ r_disc }}</cbc:Amount>
        <cbc:BaseAmount currencyID="{{ invoice.currency_code }}">{{ r_base }}</cbc:BaseAmount>
    </cac:AllowanceCharge>
    {% else %}
    <cbc:PriceAmount currencyID="{{ invoice.currency_code }}">{{ rounded(item.amount, 2) }}</cbc:PriceAmount>
    <cbc:BaseQuantity unitCode="PCE">{{ item.qty }}</cbc:BaseQuantity>
    {% endif %}
</cac:Price>
```

---

## ✅ النتيجة بعد التعديل
بعد نقل الحسبة لتتم عبر القالب (Template)، أصبحت القيم تتطابق رياضياً بشكل مثالي، وخرجت أرقام الفاتورة لزاتكا بهذا الشكل:
* السعر الأساسي (BaseAmount): `165217.39`
* الخصم (Allowance Amount): `91304.34`
* السعر الصافي المحسوب بالقالب (PriceAmount): `73913.05`

عندما تقوم زاتكا بالتدقيق: `165217.39 - 91304.34 = 73913.05` (مطابقة رياضياً بنسبة 100%).
تم **حل أخطاء الرفض تماماً**، وعملية رفع الفواتير تمر بنجاح دون أخطاء أو تحذيرات متعلقة بالعمليات الحسابية للأصناف.
