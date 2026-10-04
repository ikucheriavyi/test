# Аудит перекладів: «Legal entity registration» (Beldoc, workflow 988070)

**Джерела:**
- ТЗ `Liquio_adaptation_Beldoc_-_technical_specification_1.docx`;
- live-конфігурація `beldoc` (stg), workflow 988070, остання версія `2ea6e460-…`.

**Що перевірено:**
- задачі 988070001 (усі кроки, calculated і PDF), 002, 003, 004, 005, 006, 007;
- 14 подій-сповіщень;
- 12 подій запису статусу в реєстр (keyId 8);
- `data.statuses` і `data.timeline`;
- `toString` довідників keyId 1–6, 9, 44, 47.

**Мови:** `nl`, `eng`, `fr`. У формах ключі пишуться як `LANG_X` (форми мають `multiLanguage: true`). У PDF і сповіщеннях ключі пишуться як `{{LANG_X}}`.

> ⚠️ **Що не вдалося перевірити.** Значення LANG-ключів (сам словник) і Google Sheet «Translations» з ТЗ недоступні з середовища аудиту. Тому для кожного ключа, використаного за місцем, ми перевірили тільки наявність ключа. Значення треба звірити у словнику, список таких ключів наведено в розділі 8.

**Критичність:**
- 🔴 Висока: користувач бачить англійський текст, сирий ключ або JSON у звичайному сценарії.
- 🟠 Середня: видно в окремому сценарії, або текст змішаною мовою, або зміст розходиться з ТЗ.
- 🟢 Низька: косметичні проблеми, внутрішня роль, неконсистентність.

---

## 1. Системні проблеми, що зачіпають увесь процес

| # | Проблема | Де | Деталі | Критичність | Рекомендація |
|---|---|---|---|---|---|
| S1 | Статус заявки пишеться в реєстр англійським текстом, а не ключем | event-988070015, -039, -017, -035, -036, -023, -037, -038, -043 → `map.status` / `subStatus` | Значення: `'UNDER REVIEW'`, `'REJECTED BY CO-FOUNDER'`, `'AWAITING PAYMENT'`, `'UNDER NOTARISATION'`, `'REJECTED BY NOTARY'`, `'SENT TO BUSINESS COUNTER'`, `'WORK PERMIT REQUIRED'`. subStatus: `'Incorporation Manager review'`, `'Notary review'`. Ці значення показуються в «My applications» і «Notary dashboard» | 🔴 | Писати ключ (`LANG_UNDERREVIEW`…) або перекладати на фронті за `statusId`. Мапінг має покривати 41, 42 і 7 |
| S2 | Інші поля реєстру теж англійською | event-015/-039/-017/-020 → `coFounders[].decision`, event-015/-021 → `paymentStatus` | `'Approved'`/`'Rejected'`/`'Under review'`, `'Paid'`/`'Unpaid'`. Решта англійськими значеннями: `incorporationDateQuestion`, `whichRegion`, `addressType`, `bankAccount`, `country 'Belgium'`, `civilStatus` | 🟠 | Те саме, що S1 |
| S3 | Описи статусів захардкоджені англійською | `data.statuses[*].description` і `data.timeline.steps[*].description` (7 шт.) | Наприклад, «Your application is awaiting approval from co-founders». Label задано ключами (OK) | 🔴 | Ключі в description, якщо платформа їх підставляє, або прибрати описи |
| S4 | Усі сповіщення йдуть мовою заявника, хоч хто отримувач | `languageCode` у 13 подіях = `documents[988070001].data.languageCode` | Співзасновник (028, 003), менеджер (013), нотаріус (019, 032) і Partena (006) отримують лист мовою заявника | 🔴 | Брати мову профілю отримувача. Для Partena мову зафіксувати |
| S5 | Форми інших учасників показують дані мовою заявника | task-988070002 і -007 → `calculated.lang = doc001?.languageCode \|\| documentData?.languageCode` | NACE-коди, адреса й країна у співзасновника та нотаріуса показуються мовою заявника | 🟠 | Спершу брати `documentData.languageCode`, мову заявника лише як fallback |
| S6 | PDF генерується мовою заявника | 988070001 → `calculated.pdfLangCode`, `primaryCodeForPdf` | Співзасновники й нотаріус бачать PDF-зведення мовою заявника | 🟠 | Погодити з BA і зафіксувати в ТЗ |
| S7 | На двох формах немає `multiLanguage: true` | task-988070003 (Payment), task-988070006 (Incorporate manager) | Є в 001, 002 і 007. Без цього мова користувача в документ не пишеться | 🟠 | Додати `"multiLanguage": true` |
| S8 | Назви задач у кабінеті англійською | taskTemplate.name: «Approve the application», «Payment», «Incorporate manager Task», «Notary Task» | `jsonSchema.title` задано ключами. Але якщо «My tasks» показує `name`, користувач бачить англійську. У Notary title = `LANG_APPLICATION_INFORMATION`, а не назва задачі | 🔴 (перевірити, яке поле показує кабінет) | Перевірити на stg. Якщо показується name, перевести на ключі |

## 2. Хардкод: видимий текст без LANG-ключа

| # | Задача / крок | Елемент | Поточне значення | Очікування (ТЗ) | Критичність | Рекомендація |
|---|---|---|---|---|---|---|
| H1 | Лист Partena | event-988070006 → `subject` | `` `New сompany registration` ``, де «с» кирилична (U+0441) | «New company registration» | 🔴 | `{{LANG_NEW_COMPANY_REGISTRATION}}` (ключ уже є в тілі листа), або хоча б латинська «c» |
| H2 | Лист Partena | event-988070006 → `row('Name')`, `row('Email')` | «Name», «Email» | — | 🟠 | `{{LANG_FULL_NAME}}`, `{{LANG_CONTACTEMAIL}}` |
| H3 | 001 / крок 7 | `companyBillingEmail.checkValid[0].errorText` | «This doesn't look like email. Try it in the format example@domain.com» | Той самий текст (тільки EN) | 🔴 | `LANG_INVALID_EMAIL_FORMAT` |
| H4 | 001 / крок 2 | `companyInfo.country.value` | `() => 'Belgium'` (поле видиме, readOnly) | Belgium / Belgique / België | 🔴 | Ключ `LANG_BELGIUM` або назва з довідника 47 |
| H5 | Підпис «Email» (11 місць) | 001: `addCoFounder.items.email.description`, `founderDetails.email.description`, `founderDetails.htmlBlock`, `shareholdersInfo.table.items.email.description`, `choseNotaryInfo.email.description`. PDF 001: `<dt>Email</dt>` ×3. 002: `userDataBlock` (крок 1 і 3), `executivesInfo.email.description`. 006: `notaryEmail.description`. 007: `userDataBlock` ×2 | «Email» | Email / E-mail / E-mail | 🔴 | `LANG_EMAIL` (`{{@root.LANG_EMAIL}}` у `#each` PDF) |
| H6 | 002 / крок 3 | `executivesInfo.email.checkValid[0].errorText` | «Invalid email format.» | — | 🟠 | `LANG_INVALID_EMAIL_FORMAT` |
| H7 | 002 / крок 1 | `calculated.incorporationDateQuestion` → htmlBlock | `'As soon as possible'` / `'On a specific date'` | As soon as possible (перекладено) | 🔴 | Повертати код і рендерити `LANG_ASAP` |
| H8 | 002 / крок 1 | `calculated.whichRegion` → htmlBlock | `'Walloon Region (Wallonia)'`, `'Flemish Region (Flanders)'`, `'Brussels-Capital Region'` | FR: Région wallonne / flamande / de Bruxelles-Capitale. NL: Waals / Vlaams / Brussels Hoofdstedelijk Gewest | 🔴 | Мапити `regionCode` на LANG-ключі |
| H9 | 001 / крок 5 | `uploadFinancialPlan.metaData.description` | `` `Financial plan – ${index}${extPart}` `` | — | 🟠 | Назва файлу за languageCode |
| H10 | 001 / крок 5 + PDF; 002, 007 | `anticipatedFirstYearRevenue` items / calculated | «€0 - €25,000», «€25,000 - €100,000», «€100,000+» | FR «0 € – 25 000 €», NL «€ 0 – € 25.000» | 🟢 | Ключі для опцій |
| H11 | 001 / крок 3 | `userNameCalculated` / `userFirstNameCalculated` / `userLastNameCalculated` errorText | «Invalid user name» / «… first name» / «… last name» | — | 🟢 (поля приховані) | Ключі |
| H12 | 001 / крок 2 | `companyLanguageBlock.htmlBlock` | `{{else}}N/A{{/if}}{{else}}---{{/if}}` | — | 🟢 | Ключ або прибрати гілку |
| H13 | 001 / крок 6 | `selectYourNotary.dataMapping` | `title="Open profile" aria-label="Open profile"` | Текст посилання «European Directory of Notaries» | 🟢 | Ключ |
| H14 | 003 Payment / крок 1 | `paymentInfoBlock.htmlBlock` | `{{amount}} <span>EUR</span>`, `toFixed(2)` (2200.00) | «2 200 euros» | 🟢 | Ключ `LANG_EUROS` або погодити «EUR». Формат числа за мовою |
| H15 | Лист нотаріусу (welcome) | event-988070032 | Захардкоджений URL `https://cabinet.beci.beldoc.be` | — | 🟠 | `{{frontUrl}}` |
| H16 | Лист менеджеру | event-988070013 | Fallback `userName \|\| 'Utilisateur'` (французькою) | — | 🟢 | Прибрати |
| H17 | PDF 001 / адреса | htmlTemplate | `{{#if (eq languageCode 'eng')}}Belgium…'fr' Belgique…'nl' België` | — | 🟢 | Переклад правильний, але інлайн і без else. Краще ключ |

## 3. Помилки ключів (ризик побачити сирий `LANG_…`)

| # | Задача / крок | Елемент | Поточне значення | Проблема | Критичність | Рекомендація |
|---|---|---|---|---|---|---|
| K1 | 001 / крок 7 | `companyBillingEmail.getSample` | `LANG_REUSELANG_BILLING_EMAIL_INFO` | Склеєний ключ; імовірно відобразиться сирим | 🔴 | `LANG_BILLING_EMAIL_INFO` |
| K2 | 001 / крок 3, 002 / крок 3 | `dateOfBirth.checkValid[1]`, `dateOfMarriage.checkValid[2]` | `LANG_LANG_DATE_EARLIER_THAN` | Подвійний префікс | 🟠 | Звірити зі словником, виправити на `LANG_DATE_EARLIER_THAN` |
| K3 | 003 Payment | `paymentHint.text` | `LANG_LANG_VAT_IS_CHARGED_ADDITIONALLY_HINTTEXT` | Подвійний префікс | 🟠 | Звірити зі словником |
| K4 | 002 / крок 3 | `dateOfMarriage.checkValid[3]` | `"LANG_THEREAREFEWERDAYSINTHEMONTH."` | Крапка в кінці ключа | 🟠 | Прибрати крапку |
| K5 | 001 / крок 3 | `aboutCompanyManagerSecondHint.text` | `` `LANG_AFTER_COMPLETINGBEFORE ${n} LANG_AFTERCOMPLETING_AFTER` ``, n з `step?.executivesInfo?.addCoFounder` | Речення розбите на 2 ключі в порядку англійських слів. `step` уже є кроком executivesInfo, тож n, імовірно, завжди 0 («0 invitation(s)») | 🔴 | Один ключ із плейсхолдером; n брати з `step?.addCoFounder` |
| K6 | 001 / крок 4 | `totalShares.checkValid[2].errorText`, `table.checkValid[1].errorText` | `'LANG_WARNINGSHAREHOLDERS'`, `'LANG_THETOTALSHAREPERCENTAGE'` всередині JS-рядка в одинарних лапках | Немає чисел [actual] і [Total Shares], які вимагає ТЗ. Апостроф у FR-значенні (l'…) може зламати функцію | 🟠 | Плейсхолдери; перевірити рендер у FR |
| K7 | Лист співзасновнику | event-988070003, -028 → fullText | `${name} {{LANG_IS_CURRENTLY_REGISTERING_COMPANY}}` … `{{LANG_YOUR_REVIEW_IS_REQUIRED}} <a>{{LANG_USE_THIS_LINK}}</a> {{LANG_AND_INDICATE_WHETHER_YOU_WISH_TO_PARTICIPATE}}` | 4 фрагменти в порядку англійських слів; зміст відрізняється від ТЗ | 🟠 | Ключі під речення ТЗ (див. N3) |
| K8 | Лист «відхилено нотаріусом» | event-988070027 → subject | `{{LANG_APPLICATION_NO_PDF}} ${num} {{LANG_WAS_REJECTED_BY_NOTARY}}` | Для теми взято ключ із PDF | 🟢 | Окремий ключ теми |
| K9 | 001 / крок 2 | `thirdDescriptionSecondHint.text` | `LANG_NACECODES_CLASSIFY <a href='https://nacebel.codes/fr'>LANG_HERE</a>` | Речення розбите; URL завжди `/fr` | 🟠 | Один ключ; URL за мовою |
| K10 | 001 / крок 5 | `anticipatedHint.text` | `<a href='https://finances.belgium.be/fr/…'>LANG_MOREINFORMATION</a>` | Посилання завжди на FR-сторінку | 🟠 | URL /nl/, /fr/, /en/ за мовою |
| K11 | 001 / крок 6 | `notaryBlock.htmlBlock` | `LANG_LINTOEUNOTARYDIRECTORY` + `<a>{{notaryLink}}</a>` | Одруківка «LINTO»; замість тексту виводиться сирий URL | 🟢 | Виправити ключ, текст посилання через ключ |
| K12 | PDF 001 / Founder | `<dt>{{LANG_PARTENA_AGREEMENT}}</dt>` | Ключ чекбокса згоди | Підписом стане довге речення згоди замість «Payroll management by Partena» | 🟠 | Окремий ключ |
| K13 | PDF 001 / назва файлу | `fileName` | `const lang = documentData?.lang;` → `baseNames[lang] \|\| baseNames.EN` | Поля `lang` немає, тому назва файлу завжди англійська | 🟠 | `({eng:'EN',fr:'FR',nl:'NL'})[documentData?.languageCode]` |
| K14 | Дублікати й неконсистентність | 001 форма vs PDF; 002 vs 007 | `LANG_SHAREHOLDERS_INFORMATION` / `LANG_SHAREHOLDERINFORMATION`; `LANG_FOUNDERSMANAGEMENT` / `LANG_MANAGEMENT_AND_FOUNDERS`; `LANG_ACCOUNTANTSDETAILS` / `LANG_ACCOUNTANTDETAILS`; `LANG_VALIDEMAIL` / `LANG_INVALID_EMAIL_FORMAT`; `LANG_FIRSTFINANCIALYEAR` / `LANG_DATE_OFTHEFIRST_YEAR`; `LANG_NATIONAL_IDENTIFICATION_NUMBER` / `LANG_NISSNUMBER`; `LANG_FULLNAME` / `LANG_FULL_NAME`; `LANG_PAYMENT_PROCESSED_SUCCESSFULLY` / `LANG_YOUR_PAYMENT_HAS_BEEN_PROCESSED_SUCCESSFULLY` | Різні ключі для одного змісту | 🟢 | Уніфікувати |
| K15 | Ключі статусів | `data.statuses` | `LANG_UNDERNOTARIZATION` (z) vs NOTARISATION у ТЗ; `LANG_WORKPERMITREQUIRED` vs `LANG_WORK_PERMIT_REQUIRED_*`; `LANG_REJECTEDBYCOFOUNDER` vs `LANG_REJECTED_BY_COFOUNDER_*` | Різний стиль ключів | 🟢 | Перевірити, що у словнику саме таке написання |
| K16 | 007 / крок 4 | `bceNumber.checkValid[1]` | `LANG_ERROR_STARTS_WITH_ZERO` | Правило ТЗ: перша цифра 0 або 1 | 🟢 | Перевірити, що значення каже «0 or 1» |
| K17 | 006 / крок 1 | `applicantInfoBlock.htmlBlock` | `LANG_NOTARY_SELECTED` використано і як заголовок, і як підпис | — | 🟢 | Розвести |

## 4. Динамічні значення, що виводяться без перекладу

| # | Задача / крок | Елемент | Поточне значення | Критичність | Рекомендація |
|---|---|---|---|---|---|
| D1 | 002 / крок 1 | htmlBlock `{{#each additionalCodes}}{{this.stringified}}` | Сирий JSON `{"EN":"…","FR":"…","NL":"…"}`; primaryCode поруч виводиться правильно через `getValueByKey` | 🔴 | `{{getValueByKey this.stringified ../lang}}` |
| D2 | 001 крок 2, PDF, 002, 007, лист Partena | Регіон: довідник keyId 6, `toString: record.data.region` | Одна мова («Bruxelles-Capital», «Wallonie»…) | 🟠 | Мультимовний toString або мапінг regionCode на ключі |
| D3 | 001 крок 2, PDF | Місто: keyId 9, `record.data.place` | Одна мова (Brussel/Bruxelles) | 🟢 | Уточнити в BA |
| D4 | 007 / крок 2 | `{{founderLanguage}}` заявника | Сирі «English» / «French» / «Dutch». Для співзасновників мапиться на `LANG_*` | 🟠 | Той самий мапінг |
| D5 | 007 / крок 2, реєстр | `nationality = …nationality?.nameEng` (event-988070020) | Громадянство зберігається й показується тільки англійською, хоча keyId 47 має nameFr/nameNl | 🟠 | Зберігати код або об'єкт |
| D6 | 001 крок 6, 006 | Нотаріус: keyId 44, поля `languages` / `address` | «Dutch, French», «…, Belgium» з реєстру без перекладу | 🟠 | Коди мов, мапінг на `LANG_DUTCH` / `LANG_FRENCH`… |
| D7 | 007 / крок 1; 001 / крок 2 | NACE `register.select`: `defaultLang: "EN"`, `sortBy NATIONAL_TITLE_BE_EN` | Опції за замовчуванням англійською і відсортовані за EN. У keyId 2–4 жодне поле не `public` | 🟠 | Мова опцій = мова інтерфейсу |
| D8 | Лист Partena | event-988070006 | 'Walloon Region (Wallonia)', 'French' / 'Dutch' / 'English', 'Belgium'; alreadySelfEmployed без мапінгу. Тіло листа йде через ключі, тема й значення англійською, тож мова змішана | 🟠 | Узгодити мову листа з BA (див. S4) |
| D9 | 001 / крок 3 | Рядок ролі `{{#if (eq isManager 'yes')}}LANG_MANAGER{{else}}LANG_FOUNDER{{/if}}` | isManager — масив `["yes"]`, тож умова, імовірно, завжди хибна, і директор підписаний як Founder | 🟠 | `includes` |
| D10 | 001 / крок 4 | spreadsheet.lite: кнопки Import / Clear | Тексти платформи | 🟢 | Перевірити nl/fr або вимкнути |
| D11 | Реєстр, event-988070039 | Мапінг `companyLanguage` | Немає `english`, тож після повторного подання 'English' стає 'Dutch' | 🟠 | Додати `english: 'English'` |

## 5. Невідповідність ТЗ у змісті сповіщень, фінальних екранів і статусів

| # | Де | Поточне | Очікування за ТЗ (EN / FR / NL) | Критичність | Рекомендація |
|---|---|---|---|---|---|
| N1 | Статус AWAITING PAYMENT | Прив'язано до event-988070012 (до gateway «Is application paid?»), тож ставиться й оплаченим | «approved by all co-founders, but not paid yet» | 🟠 | Прив'язати до event-988070035 |
| N2 | Статус UNDER REVIEW | Прив'язано до task-988070004, яка виконується лише коли є співзасновники; опис «…awaiting approval from co-founders» | Ставиться для всіх після «Finish» | 🟠 | Прив'язати до event-988070018 |
| N3 | Лист співзасновнику (003, 028) | `{{LANG_HELLO}}!` без 👋 та імені (approverName обчислюється після `return`, тобто мертвий код), тіло з 4 фрагментів | «👋 Dear, <CO-FOUNDER'S NAME>!» + EN «The application submitted by <Applicant's Name> is now awaiting your approval…» / FR «La demande soumise par <…> est actuellement en attente de votre approbation…» / NL «De aanvraag ingediend door <…> wacht momenteel op uw goedkeuring…» | 🟠 | Привести до ТЗ |
| N4 | Лист «відхилено нотаріусом» (027) | `{{LANG_YOUR_APPLICATION}} n {{LANG_WAS_REJECTED_BY_NOTARY}}` + `{{LANG_YOU_CAN}} <a>{{LANG_EDIT_THE_APPLICATION}}</a>` | «…You can edit the application and send it for re-approval.» / «Vous pouvez modifier la demande et la soumettre à nouveau pour approbation.» / «U kunt de aanvraag aanpassen en opnieuw ter goedkeuring indienen.» | 🟠 | Може бракувати «send it for re-approval»; краще один ключ-речення |
| N5 | Лист Partena | `{{LANG_SHAREHOLDER_SINGULAR}}` без номера; Net remuneration завжди `LANG_MINIMUM_SALARY`; VAT завжди `LANG_YES`; Business takeover завжди `LANG_NO`; ключі з «BECI» в назві | «Shareholder N», «€2,500», Yes/No за даними | 🟢 | Узгодити з BA |
| N6 | 003 Payment / фінальний екран | Кнопка `LANG_TO_RECEIVED_MESSAGES` → `/messages` | Кнопка Home / «Aux services demandés» / Home | 🟠 | Кнопка Home з ключем |
| N7 | 006 / фінальний екран | Одна кнопка `LANG_TO_NOTARY_DASHBOARD`, `showNextTaskButton:false` | «Next task» + «Cabinet» | 🟢 | Узгодити |
| N8 | 007 / фінальний екран | Subtitle `LANG_DECISION_RECORDED_SUBTITLE` для обох рішень; `LANG_APPLICATION_APPROVED_TITLE` без тексту в ТЗ | Reject: «Application rejected» / «Demande rejetée» / «Aanvraag afgewezen» + «Your decision has been recorded» / «Votre décision a été enregistrée» / «Uw beslissing werd geregistreerd» | 🟢 | Погодити текст для Approve |
| N9 | 003 Payment / крок 1 | VAT-хінт без `checkHidden`, показується завжди | Показувати тільки після «Back» зі Stripe | 🟢 | Додати умову |
| N10 | 002 / крок 3 | `payrollManagementByPartenaHint.checkHidden` читає `doc?.executivesInfo?.addMyselfAsFounder[0]…` (шлях із форми 001). Буде TypeError, і банер показується завжди | Банер лише коли чекбокс не відмічено | 🟠 | `(v, step) => step?.payrollManagementByPartena?.includes('Yes')` |
| N11 | 001 / крок 5 | Окрім потрібного хінта, є зайві `LANG_BANKACCOUNTLATERHINT` і infobox `LANG_OFFICIALCOMPANYBANKACCOUNT` / `LANG_OFFICIALBANK_LATER` | Лише «The company's official bank account will be used for tax and VAT refunds…» | 🟢 | З'ясувати зміст |
| N12 | Назви кроків 001 | `LANG_MANAGEMENT_AND_FOUNDERS`, `LANG_NOTARIZATIONPROCESS`… | «Founders & Management», «Notarisation Process» | 🟢 | Звірити значення (порядок слів, z/s) |
| N13 | Статус LANG_REJECTEDBYCOFOUNDER, description | «…by one or more co-founder» | co-founders | 🟢 | Виправити (див. S3) |

## 6. Відсутні елементи (тексту немає, тож немає й перекладу)

| # | Задача / крок | Чого немає | Текст у ТЗ | Критичність |
|---|---|---|---|---|
| M1 | 001 / крок 1 | Radio «Is the company being formed through a business takeover?» (Yes/No) | Тільки EN | 🔴 |
| M2 | 001 / крок 1 | Перевірка назви в CBE і помилка «A company with this name already exists in the Crossroads Bank for Enterprises: <…>. Change your company name» | Тільки EN | 🔴 |
| M3 | 001 / крок 5 | Блок «First Financial Year End»: радіо «Short year - December 31, <рік>» / «Long year - December 31 <рік+1>» і підказки до них | Тільки EN | 🔴 |
| M4 | Сповіщення AWAITING PAYMENT | event-988070010 «Notification» має порожній `jsonSchema: {}` | «Your application has been approved. Please proceed with the payment» / «Votre demande a été approuvée. Veuillez procéder au paiement» / «Uw aanvraag werd goedgekeurd. Gelieve verder te gaan met de betaling» | 🟠 |
| M5 | 003 Payment | Поле «Payment amount» і текст «You can try to pay again by refreshing the page in 5 minutes» | Тільки EN | 🟠 |
| M6 | 001 / крок 3 | Попап «Adding co-founder» і хінт «Edit the distribution of shares in step 4…» | Тільки EN | 🟠 |
| M7 | 001 / крок 3 | Валідація віку ≥18 та її текст | Текст не задано | 🟠 |
| M8 | 001 / крок 5 | «Supported files: PDF and Excel (.xls, .xlsx)», «Maximum file size: 25 MB» | Тільки EN | 🟠 |
| M9 | 002 / крок 1, 007 / кроки 1 і 3 | Значення «Not provided» (вебсайт, фінплан) і «As soon as possible» у Notary | «Not provided» / «As soon as possible» | 🟠 |
| M10 | 007 / крок 1 | Не виводяться additionalCodes (NACE) | — | 🟢 |
| M11 | 002 / крок 3 | Тексти помилок NIN (min 5 / max 30) і Location of marriage (min 10 / max 100) | — | 🟢 |
| M12 | Статуси 41 і 42 | Немає в statuses / timeline | Без FR/NL | 🟢 |

## 7. Помилки в самому ТЗ (переклади та термінологія)

| # | Розділ ТЗ | Проблема | Як виправити | Критичність |
|---|---|---|---|---|
| T1 | Крок 3, блокер sole director | **Порожній NL** («NL:» без тексту). EN: «is foreign national» без артикля | Додати NL; «is a foreign national» | 🔴 |
| T2 | Статус REJECTED BY NOTARY, Subject | FR «Rejeté par le notaire» і NL «Afgewezen door de notaris» без номера заявки і без demande/aanvraag; «NL:Afgewezen» без пробілу | «La demande <n> a été rejetée par le notaire» / «Aanvraag <n> werd afgewezen door de notaris» | 🟠 |
| T3 | REJECTED BY NOTARY, Text | У FR/NL немає `<Reason>`; одруківка NL «Uw **anvraag**» | Додати `<Reason>`; «aanvraag» | 🟠 |
| T4 | UNDER REVIEW | «FR:EN COURS DE TRAITEMENTNL: IN BEHANDELING»: FR і NL злиплися. «👋 Dear, <NAME>!» є тільки в ENG. EN без крапок, FR/NL з крапками. FR Message 2 довший («afin de poursuivre le processus») | Розділити; додати привітання FR/NL; вирівняти | 🟠 |
| T5 | Message 3 (незареєстрований співзасновник) | Текст = Message 2; мова листа не визначена | Визначити мову | 🟠 |
| T6 | WORK PERMIT REQUIRED | Немає в таблиці статусів (ID 7, FR/NL назва); тема «Work permit for director» тільки EN | Додати | 🟠 |
| T7 | AWAITING PAYMENT, UNDER NOTARISATION, SENT TO BUSINESS COUNTER | Немає Where to send / Recipient / Subject | Доповнити | 🟠 |
| T8 | Сповіщення без тексту | Нотаріусу про нову задачу, менеджеру, про оплату (011), заявнику без співзасновників (`LANG_APPLICATION_SUCCESSFULLY_SUBMITTED_NOTIFICATION`) | Додати EN/FR/NL | 🟠 |
| T9 | Welcome-лист нотаріусу, лист Partena | Тільки EN; «BelDoc» vs «Beldoc»; у Partena статус «Ready for notarisation» суперечить ID 6; «value form the field»; «RNOKPP (formerly TIN)» (український ідентифікатор) | Виправити; визначити мову | 🟠 |
| T10 | Payment, фінальний екран | FR-кнопка «Aux services demandés» проти EN/NL «Home». У FR/NL є «prochainement/binnenkort» і «e-mail de confirmation», яких немає в EN | Вирівняти (FR «Accueil») | 🟠 |
| T11 | Кроки форм (усі задачі) | Підписи, опції, хінти, помилки лише EN, ключі не вказані; переклади тільки в Google Sheet | Додати відповідність текст ↔ ключ | 🟠 |
| T12 | My applications: Decision / Role / Payment status | Значення лише EN, вимоги перекладу немає | Додати | 🟠 |
| T13 | REJECTED BY CO-FOUNDER | Значення `<Decision N>` не визначені; FR «REJETÉ PAR LE COFONDATEUR / NOTAIRE»: рід і число (la demande → «REJETÉE») | Approved/Rejected · Approuvé/Rejeté · Goedgekeurd/Afgewezen; перевірити перекладачем | 🟢 |
| T14 | Термінологія | «EN:» / «ENG:»; Notarisation / NOTARIZATION; Director vs Manager (`LANG_MANAGER`); «Founders & Ownership» vs «Founders & Management»; «Initial contribution amount» vs «Share Capital Amount»; «Bank account» vs «Business Bank Account»; «Notary task» / «Notarisation» | Уніфікувати | 🟢 |
| T15 | Дрібні одруківки | «#Emal»; «city and **county**» (marriage); «mm/dd/yyyy. Example: 03.15.1990» (а має бути dd.mm.yyyy); «financial year-end fiscal year»; «Bruxelles-Capital» (має бути «Bruxelles-Capitale»); подвійний пробіл «registered.  The» (перенесено в конфіг); незакриті лапки; «one of three options» при двох; два варіанти хінта кроку 6; порожні Name і Type банера фінплану; типи полів українською в Incorporation manager | Виправити | 🟢 |

## 8. Ключі є, але значення треба звірити зі словником (nl / eng / fr)

Ключі використано правильно, проте ТЗ дає точний текст, і значення ключа в словнику треба порівняти з ним дослівно:

| Ключ(і) | Де | Очікуваний текст за ТЗ |
|---|---|---|
| `LANG_SOLE_DIRECTOR_WORK_PERMIT_REQUIRED` | 001 / крок 3, `executivesInfo.checkValid[1]` | EN і FR з ТЗ, 2 абзаци; NL у ТЗ немає (T1) |
| `LANG_WORK_PERMIT_REQUIRED_SUBJECT/TEXT/HINT` | event-988070044 | «Work permit for director» + EN/FR/NL «All directors are foreign nationals…» |
| `LANG_APPLICATIONSUCCESSFULLYSUBMITTED`, `LANG_HELLO`, `LANG_PLEASE_WAIT_FOR_CONFIRMATION_FROM_ALL_CO_FOUNDERS` | event-988070018 | «Application successfully submitted» / «Demande soumise avec succès» / «Aanvraag succesvol ingediend»; «Dear» чи «Hello»? |
| `LANG_APPLICATION_PENDING_APPROVAL_SUBJECT` | event-003, -028 | «Application pending your approval» / «Demande en attente de votre approbation» / «Aanvraag wacht op uw goedkeuring» |
| `LANG_REJECTED_BY_COFOUNDER_SUBJECT_BEFORE/AFTER`, `…_EDIT_HINT_PREFIX/LINK`, `LANG_APPROVED`, `LANG_REJECTED` | event-988070024 | «Application <n> was rejected» / «La demande <n> a été rejetée» / «Aanvraag <n> werd afgewezen» |
| `LANG_APPLICATION_SUCCESSFULLY_SENT_TO_NOTARY_SUBJECT/TEXT` | event-988070007 | «Your application has been successfully sent to the notary…» / «Votre demande a été envoyée avec succès au notaire…» / «Uw aanvraag werd succesvol naar de notaris verzonden…» |
| `LANG_LEGAL_ENTITY_APPROVED_TITLE/MESSAGE` | event-988070026 | «Your legal entity has been registered…» / «Votre entité juridique a été enregistrée…» / «Uw juridische entiteit werd geregistreerd…» |
| 7 label-ключів статусів | `data.statuses` | UNDER REVIEW / EN COURS DE TRAITEMENT / IN BEHANDELING; AWAITING PAYMENT / EN ATTENTE DE PAIEMENT / IN AFWACHTING VAN BETALING; UNDER NOTARISATION / EN COURS DE NOTARISATION / IN NOTARIËLE VERWERKING; REJECTED BY NOTARY / REJETÉ PAR LE NOTAIRE / AFGEWEZEN DOOR DE NOTARIS; SENT TO BUSINESS COUNTER / ENVOYÉ AU GUICHET D'ENTREPRISE / VERZONDEN NAAR HET ONDERNEMINGSLOKET; REJECTED BY CO-FOUNDER / … |
| `LANG_DECISION_APPROVED_SUBTITLE`, `LANG_DECISION_REJECTION_SUBTITLE`, `LANG_APPLICATION_TITLE` | 002 / фінальний екран | «Uw beslissing om (niet) deel te nemen werd geregistreerd…» тощо |
| `LANG_THANKYOU`, `LANG_YOUR_PAYMENT_HAS_BEEN_PROCESSED_SUCCESSFULLY` | 003 / фінальний екран | Обидва речення: «…processed successfully. You will receive an email…» |
| `LANG_NOTARY_APPOINTED`, `LANG_NOTARY_EMAIL_INFO` | 006 / фінальний екран | «The notary has been appointed» / «The notary will receive the welcome message…» |
| `LANG_APPLICATION_REJECTED_TITLE`, `LANG_DECISION_RECORDED_SUBTITLE` | 007 / фінальний екран | Див. N8 |
| `LANG_WEWILLNOT`, `LANG_ADDRESSASSOON`, `LANG_FULLNAME` (getSample), `LANG_LLC` | 001 / крок 1 | Назви ключів не відповідають змісту банерів. `LANG_LLC` має бути SRL (FR) і BV (NL) |
| `LANG_MANAGER`, `LANG_ABOUTCOMPANYMANAGERS`, `LANG_COMPANY_MANAGERS_HAVE` | 001, 002, 007 | Director / Administrateur / Bestuurder |
| `LANG_SPECIFYCITYANDCOUNTYOFYOURMARRIAGEREGISTRATION`, `LANG_MARRIAGE_CITY_COUNTRY` | 001, 002 | «city and country» (не «county») |
| `LANG_PLEASE_MAKE_SURE_ALL_INFORMATION_IS_ACCURATE_TO_HELP_THE_NO` | 001 / крок 3 | Обрізана назва ключа |

## 9. Поза перекладами (помічено під час аудиту)

- **Відносні посилання.** event-024 і -027 мають `{{frontUrl}}tasks/…` без «/», а інші події `{{frontUrl}}/tasks/…`. Одна з двох форм дає биту адресу.
- **`formatName` без захисту від порожнього значення.** У -018, -007, -026, -019 він викликає `str.toLowerCase()` без `?.`. Якщо ім'я порожнє, сповіщення не надсилається.
- **Неправильна умова в event-988070015.** Він записує `companyAddress` за умовою `incorporationDateQuestion` замість `addressType`.
- **Захардкоджені отримувачі в event-988070006.** Розсилка йде на прод-адреси Partena і Beldoc разом із тестовими адресами kitsoft.
- **Зайві welcome-листи.** event-988070032 надсилається нотаріусу для кожної заявки, навіть якщо нотаріус уже зареєстрований.
