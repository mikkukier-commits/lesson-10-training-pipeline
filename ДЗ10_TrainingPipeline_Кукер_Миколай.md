# ДЗ10. Automated Training Pipeline: контроль процесу, ризики та production-readiness

## 1. High-level workflow automated training pipeline

### Схема процесу

```text
GitLab event / manual trigger
          ↓
GitLab CI pipeline
          ↓
AWS Step Functions execution
          ↓
Data validation
          ↓
Training job
          ↓
Model evaluation
          ↓
Logging metrics and artifacts (MLflow)
          ↓
Model Registry update (candidate)
          ↓
Notification to team
          ↓
Manual review / approval before release
```

### Ролі компонентів і логіка процесу

| Етап | Призначення | Контроль процесу |
|---|---|---|
| GitLab event / manual trigger | Запускає процес після merge у `main`, вручну або за узгодженим розкладом | Перевіряються дозволений тип запуску, branch і зміни в коді/config |
| GitLab CI | Перевіряє код і конфігурацію, передає параметри запуску та ініціює AWS Step Functions | Доступ до AWS обмежений, Git-контекст передається в execution input |
| AWS Step Functions | Оркеструє послідовність кроків, залежності, retries і обробку помилок | Execution має видимий статус; помилки не повинні непомітно обходити quality gates |
| Data validation | Перевіряє схему, обов’язкові поля, типи, пропуски та допустимі діапазони | За критичної помилки training не запускається |
| Training job | Навчає модель на визначеній версії даних і коду | Фіксуються параметри, версії та статус job |
| Model evaluation | Обчислює погоджені ML-метрики та порівнює результат із baseline / production-моделлю | Модель, яка не пройшла критерії, не просувається далі |
| Logging | Зберігає метрики, параметри, логи та артефакти в MLflow або погодженій системі | Результат має бути пов’язаний із конкретним execution і commit |
| Model Registry | Реєструє модель-кандидат і її версію після успішних перевірок | Реєстрація candidate не означає автоматичне схвалення production |
| Notification | Повідомляє команду про success/failure, метрики та наступну дію | Повідомлення містить посилання на execution/logs і відповідального |

**GitLab CI** відповідає за стартову перевірку та запуск workflow, а не за оркестрацію кожного ML-кроку. **AWS Step Functions** координує кроки workflow та їхні переходи. Метрики, параметри й артефакти фіксуються в **MLflow** (або іншому затвердженому tracking-сервісі). Модель-кандидат і її версія зберігаються в **Model Registry**. Повідомлення отримують AI/ML Engineer, MLOps/DevOps, AI PM або Product Owner і Tech Lead; конкретний список каналів та отримувачів має бути погоджений командою.

Технічний статус `SUCCEEDED` означає, що workflow завершився без технічної помилки. Він **не** доводить, що модель має достатню якість або готова до production.

## 2. Контрольні точки pipeline

| Етап pipeline | Що перевіряє команда | Owner | Критерій успіху | Дія у разі помилки |
|---|---|---|---|---|
| Trigger і Git-контекст | Дозволений trigger, branch, commit SHA, source та конфігурація | DevOps / MLOps Engineer | Execution містить коректні ідентифікатори запуску | Не запускати training; перевірити правила CI та input |
| Data validation | Наявність даних, schema, типи, обов’язкові поля, пропуски й допустимі діапазони | Data Engineer | Усі критичні перевірки пройдені | Зупинити pipeline, повідомити Data Engineer, не запускати training |
| Training job | Версії коду й даних, параметри, ресурси та статус job | AI/ML Engineer | Job завершився, параметри й версії записані | Перевірити логи; виправити причину; повторити лише після усунення проблеми |
| Model evaluation | Погоджені метрики, baseline comparison і ключові сегменти | AI/ML Engineer; AI PM перевіряє бізнес-критерії | Усі обов’язкові quality gates пройдені | Позначити як rejected/failed quality gate; не просувати модель |
| Logging / artifacts | Наявність метрик, параметрів, логів, артефактів і зв’язку з commit/execution | MLOps Engineer | Результати доступні та однозначно пов’язані з запуском | Не реєструвати модель як придатну до release; відновити logging і повторити необхідний крок |
| Model Registry | Версія моделі, metadata, походження, статус і посилання на evaluation | MLOps Engineer | Модель зареєстрована як `candidate` лише після успішних gates | Перевірити права, registry та metadata; не виконувати release |
| Approval / release decision | Результати gates, ризики, бізнес-критерії та план відкату | Tech Lead / Model Owner; AI PM підтверджує бізнес-критерії | Є задокументоване рішення уповноваженої особи | Залишити candidate без production-релізу |
| Notification | Статус, причина помилки, метрики, посилання та наступна дія | MLOps Engineer; AI PM контролює зрозумілість повідомлення | Відповідальні отримали actionable notification | Повторити notification або використати погоджений резервний канал |

### Відповідальність і ескалація

- **Data Engineer:** виправлення проблем даних і data validation.
- **AI/ML Engineer:** тренування, evaluation, інтерпретація метрик і технічний аналіз невдалих запусків.
- **MLOps/DevOps Engineer:** GitLab CI, AWS permissions, Step Functions, logging, Registry integration і alerts.
- **Tech Lead / Model Owner:** технічний review та рішення щодо проходження технічних gates.
- **AI PM / Product Owner:** бізнес-критерії, координація рішення та інформування стейкхолдерів.
- **Release approver:** уповноважений Tech Lead або Model Owner, визначений командою; production promotion не має відбуватися автоматично без цього approval.

Пропоноване правило ескалації: owner починає первинний аналіз у межах **4 робочих годин** після alert. Якщо проблему не локалізовано протягом **1 робочого дня**, її передають Tech Lead / MLOps lead і фіксують блокер. Для підозри на витік секретів, несанкціонований AWS-доступ або неконтрольовані production-зміни ескалація відбувається негайно. Це запропоновані SLA, які команда має підтвердити.

## 3. Quality gates для нової моделі

Конкретні числові пороги залежать від задачі, baseline і бізнес-вимог. Їх потрібно погодити до реалізації pipeline, а не підбирати після отримання результату. Нижче наведено приклад правил; це не універсальні ML-пороги.

| Quality gate | Правило / що перевіряється | Хто перевіряє | Рішення |
|---|---|---|---|
| Data quality | Немає критичних помилок schema; обов’язкові поля та типи коректні; data checks пройдені | Data Engineer | Продовжити або зупинити pipeline |
| Reproducibility / lineage | Зафіксовані commit SHA, версія даних, конфігурація, seed (якщо застосовно) та параметри запуску | AI/ML Engineer + MLOps | Продовжити або заблокувати candidate |
| ML metric threshold | Основна метрика (наприклад, F1-score) не нижча за заздалегідь погоджений threshold | AI/ML Engineer | Продовжити або відхилити |
| Baseline comparison | Основна метрика не гірша за поточну production-модель / погоджений baseline; зміни інших метрик також перевіряються | AI/ML Engineer | Продовжити або відхилити / відправити на аналіз |
| Segment / business checks | Немає неприйнятної деградації на ключових сегментах; результат відповідає бізнес-вимогам | AI PM / Product Owner + AI/ML Engineer | Погодити або повернути на аналіз |
| Artifacts and Registry | Метрики й артефакти доступні; модель має lineage та зареєстрована зі статусом `candidate` | MLOps Engineer | Продовжити review або заблокувати |
| Manual approval | Tech Lead / Model Owner перевірив результати та зафіксував рішення | Tech Lead / Model Owner; AI PM підтверджує бізнес-критерії | Дозволити staging/release review або залишити candidate |
| Production release gate | Staging/smoke tests пройдені, є monitoring і rollback plan, отримано окремий production approval | Release approver + MLOps/SRE | Дозволити release або заблокувати |

**Пороги до погодження:** команда має визначити основну метрику, мінімальне значення, baseline, допустиме відхилення на ключових сегментах і вимоги до тестового набору. Наприклад, можна встановити правило `F1-score ≥ погодженого threshold` та `F1-score не нижче production baseline`, але конкретні числа потрібно визначати для конкретного продукту.

Якщо training завершився успішно, але модель не пройшла хоча б один обов’язковий quality gate, її не просувають. Результат зберігають для аналізу, модель позначають як таку, що не пройшла evaluation / gates, а команда отримує повідомлення з причиною. Повторний запуск можливий після виправлення даних, коду або конфігурації; пороги не можна послаблювати лише для того, щоб модель пройшла перевірку.

## 4. Правила запуску training pipeline

| Trigger | Коли доречний | Перевага | Ризик | Рекомендоване середовище та контроль |
|---|---|---|---|---|
| Push / merge у `main` | Після зміни training code або pipeline config | Раннє виявлення проблем, відтворюваний процес | Зайві запуски й витрати через нерелевантні зміни | Автоматично для dev; використовувати CI rules, перевірки змінених файлів і ліміти concurrency |
| Manual trigger | Для експерименту, повторного запуску, staging evaluation або контрольованого retraining | Є контроль над часом і причиною запуску | Залежність від людини; ризик запуску без потрібних параметрів | Dev/staging; production-oriented run лише уповноваженою роллю з перевіркою параметрів і audit trail |
| Schedule | Регулярне retraining за погодженою частотою | Передбачуваність і менше ручної роботи | Може тренувати без потреби; можливі зайві cloud costs | Staging спочатку; для production — лише за обґрунтованою частотою, budget limit і quality gates |
| Зміна model config | Після зміни гіперпараметрів, feature selection або training config | Перевіряє вплив конкретної зміни | Неперевірені зміни можуть спричинити регресію | Dev; зміни конфігурації мають проходити code review і бути версійованими |
| Drift / quality degradation signal | Коли monitoring показує погоджений drift або деградацію | Запуск пов’язаний із реальним сигналом | Хибні спрацювання, шум або непомічена проблема без зрілого monitoring | Staging evaluation спочатку; production retraining тільки після перевірки сигналу, evaluation і approval |

**Рекомендована стартова політика:** автоматичні перевірки коду/config і відповідні training runs у dev; контрольований запуск у staging після релевантної зміни або за розкладом; production promotion лише після quality gates і manual approval. Drift-сигнал сам по собі не повинен автоматично замінювати production-модель. Усі запуски мають мати причину, owner-а, Git-контекст і ліміти ресурсів/паралельності.

## 5. Risk register

Шкала: **Low / Medium / High**. Оцінки нижче є початковою якісною оцінкою для планування; їх слід переглянути з командою з урахуванням архітектури та критичності продукту.

| Ризик | Ймовірність | Вплив | Mitigation / контроль | Owner |
|---|---|---|---|---|
| 1. GitLab CI має надмірні AWS-права | Medium | High | Least privilege; окремі ролі для dev/staging/prod; обмежити дозволені actions і ресурси; регулярно переглядати permissions | DevOps / Cloud Security |
| 2. AWS credentials або secrets витікають у logs/variables | Medium | High | Не зберігати ключі в репозиторії; використовувати OIDC/короткоживучі credentials, masked/protected variables, secret scanning і ротацію за потреби | DevOps / Security |
| 3. Неавторизований запуск або production promotion | Medium | High | Protected branches/environments, обмеження ролей, approval rules, audit trail; заборонити автоматичний production release без approval | DevOps + Release approver |
| 4. Дані мають неправильну схему або низьку якість | Medium | High | Data validation до training; versioning; чіткі правила для nulls, типів, діапазонів і schema changes; зупиняти workflow при критичній помилці | Data Engineer |
| 5. Модель не проходить quality gates або гірша за baseline | Medium | High | Зафіксувати метрики й thresholds до запуску; baseline comparison; сегментні перевірки; candidate не просувати без approval | AI/ML Engineer + AI PM |
| 6. Training запускається занадто часто, зростають витрати | High | Medium | Rules by changed files; schedule замість зайвих тригерів; concurrency limits; budgets/alerts; автоматичне завершення завислих jobs | MLOps / FinOps owner |
| 7. Pipeline падає через timeout, AWS quota або тимчасову помилку | Medium | Medium/High | Визначити retry policy для transient errors, timeout і retry limits; logs; idempotency; alert після остаточної невдачі; documented runbook | MLOps Engineer |
| 8. Метрики або артефакти не записалися / не відповідають execution | Medium | High | Перевіряти наявність обов’язкових artifacts і метрик; пов’язувати run з commit SHA та Step Functions execution ARN; блокувати promotion без повного lineage | MLOps Engineer |
| 9. Повторний retry створює дубльовані записи або небажані дії | Medium | Medium | Ідемпотентні кроки, унікальний run/execution ID, визначена retry policy та перевірка перед повторною реєстрацією моделі | MLOps Engineer |
| 10. Notification не доставлено або не містить потрібного контексту | Medium | Medium | Перевіряти доставку, додати резервний канал, runbook і посилання на logs; у повідомленні вказувати owner та наступну дію | MLOps Engineer + AI PM |
| 11. Залежності або базовий образ змінюються без контролю | Medium | High | Lock dependencies / versioned images; code review; vulnerability scanning; повторюване середовище training | AI/ML Engineer + DevOps |
| 12. Registry candidate помилково сприймають як production-approved | Medium | High | Окремі статуси candidate/approved/production; документація lifecycle; promotion permissions і manual approval | Model Owner / MLOps |

Перед production використанням команда має підтвердити власників ризиків, налаштування доступу, бюджетні обмеження, alerting, retry policy, quality thresholds та процедуру incident response.

## 6. Acceptance criteria для Jira-задачі

**Задача:** «Інтегрувати GitLab CI з AWS Step Functions для automated training pipeline».

Задачу можна вважати виконаною, коли всі наведені нижче критерії перевірені:

- [ ] GitLab CI може запустити визначений AWS Step Functions state machine через погоджений механізм автентифікації.
- [ ] У Step Functions input передаються `commit_sha`, `branch` і `source="gitlab-ci"`.
- [ ] У запуску збережені унікальний pipeline/run identifier та посилання на відповідний GitLab pipeline.
- [ ] Уповноважений член команди може знайти execution у AWS Console і побачити його статус, тривалість та історію кроків.
- [ ] Workflow має окремий `data validation` step, який виконується до training.
- [ ] Якщо data validation завершується критичною помилкою, training не запускається, а статус і причина помилки доступні команді.
- [ ] Training step використовує визначені версії коду, даних і конфігурації; вони доступні для аудиту.
- [ ] Evaluation обчислює погоджені ML-метрики та порівнює результат із baseline / production-моделлю.
- [ ] Quality gates використовують заздалегідь погоджені thresholds; невиконання gate блокує просування моделі.
- [ ] Метрики, параметри, логи й потрібні артефакти записуються в MLflow або іншу погоджену систему tracking.
- [ ] Tracking record можна пов’язати з commit SHA, GitLab pipeline і Step Functions execution.
- [ ] Model Registry оновлюється лише після успішних обов’язкових перевірок; нова модель отримує статус `candidate`, а не автоматично `production`.
- [ ] Перехід у staging/production має визначені approval rules; production release неможливий без окремого manual approval від уповноваженого approver-а.
- [ ] GitLab CI використовує least-privilege доступ до AWS; секрети не зберігаються в коді й не виводяться у відкриті logs.
- [ ] Для dev/staging/prod визначені дозволені ролі та межі доступу; критичні production actions доступні лише уповноваженим ролям.
- [ ] Для transient failures визначено retry/timeout policy; після остаточного failure pipeline завершується зрозумілим статусом і не створює неузгодженого release.
- [ ] Команда отримує notification про success і failure з назвою pipeline, commit/branch, статусом, ключовими метриками або причиною помилки, посиланнями та наступною дією.
- [ ] Для невдалого запуску доступний runbook або задокументована процедура: хто аналізує проблему, коли ескалювати та за яких умов повторювати запуск.
- [ ] Визначені concurrency/resource limits або інший контроль, що запобігає неконтрольованим повторним запускам і витратам.
- [ ] У документації пояснено, що `SUCCEEDED` у Step Functions означає технічне завершення workflow, але не автоматичну production-readiness моделі.

## 7. Notification messages

Наведені нижче назви моделі, commit-и та метрики — приклади формату, а не фактичні результати запуску.

### 7.1. Success notification

**[SUCCESS] Automated training pipeline completed**

- **Pipeline:** `churn-model-training`
- **Status:** Step Functions execution `SUCCEEDED`
- **Branch / commit:** `main` / `a1b2c3d`
- **Source:** `gitlab-ci`
- **Execution:** AWS Step Functions → `execution-123` (посилання на execution)
- **Evaluation:** F1-score `0.83`; Recall `0.79`
- **Baseline:** пройдено / не пройдено (вказати фактичний результат)
- **Quality gates:** passed / failed (вказати фактичний результат)
- **Model Registry:** `candidate` лише якщо всі обов’язкові gates пройдено
- **Details:** посилання на GitLab pipeline, MLflow run, evaluation report та Registry
- **Next action:** Tech Lead / Model Owner перевіряє результати й ухвалює рішення щодо наступного етапу; production release потребує окремого approval
- **Owner:** Tech Lead / Model Owner; AI PM координує бізнес-review

### 7.2. Failure notification

**[FAILURE] Automated training pipeline failed**

- **Pipeline:** `churn-model-training`
- **Status:** `FAILED`
- **Branch / commit:** `feature/retrain` / `a1b2c3d`
- **Source:** `gitlab-ci`
- **Failed step:** `DataValidation`
- **Reason:** наприклад, відсутня обов’язкова колонка `customer_age` (замінити фактичною причиною)
- **Execution:** AWS Step Functions → `execution-123` (посилання на execution)
- **Details:** посилання на GitLab job logs, Step Functions execution history та data validation report
- **Next action:** Data Engineer перевіряє dataset/schema; повторний запуск дозволено після виправлення причини
- **Owner:** Data Engineer
- **Escalation:** якщо проблему не локалізовано протягом 4 робочих годин, повідомити Tech Lead / MLOps lead; підозра на security incident ескалується негайно

Notification має формуватися з фактичних даних execution. Не можна показувати прикладові метрики як реальні результати або позначати gates як passed, якщо перевірки не виконалися.

## 8. Короткий висновок

Automated training pipeline робить тренування моделей повторюваним, контрольованим і простішим для аудиту. GitLab CI перевіряє контекст запуску та ініціює workflow, а AWS Step Functions координує його етапи й обробку помилок. MLflow або інша tracking-система зберігає метрики, параметри та артефакти, а Model Registry фіксує версію і статус моделі-кандидата. Технічний успіх pipeline не гарантує якості прогнозів і не означає готовності до production. Перед просуванням модель має пройти data validation, evaluation, порівняння з baseline, бізнес-перевірки та погоджені quality gates. Рішення про production має ухвалювати уповноважений approver після review, а не бути автоматичним наслідком успішного training. Контроль доступів, витрат, сповіщень, retry policy та відповідальності допомагає команді реагувати на проблеми й довіряти процесу, а не лише статусу `SUCCEEDED`.
