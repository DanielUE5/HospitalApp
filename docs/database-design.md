# Проект на база данни за няколко болници

Статус: схема за преглед преди C# моделите и първата миграция. Финансовите политики, необходими преди реална експлоатация, са описани в [плана](development-plan.md).

Връзки: [обхват](project-scope.md), [концепция](multi-hospital-proposal.md), [архитектурни източници](database-research.md), [ценово проучване](pricing-research-bg.md).

## Принципи

- Общи за платформата са акаунтът, самоличността, Patient, Doctor и каталогът Specialties.
- Всички болнични записи имат HospitalId. Кратките диаграми показват основните отношения; таблиците и ограниченията уточняват принадлежността.
- Основните ключове са UUID/Guid. Свързващите таблици могат да имат съставен ключ.
- DateOfBirth е date; събитията са timestamptz (UTC); сумите са numeric(18,2) плюс Currency. „?“ означава nullable.
- Възраст, свободни легла и текущо платено/остатък се изчисляват.
- Медицинските файлове са частни; базата пази метаданни и ключ към хранилището, не публичен URL.
- Одобрената оферта, разпределението на легло, реалният престой и платежните операции са различни обекти.
- Каталогът и наличността във v1 се управляват през нашия болничен панел. Външните интеграции са бъдещ етап.

## 1. Акаунти, болници и роли

~~~mermaid
erDiagram
    PERSON ||--|| APPLICATION_USER : login
    PERSON ||--o| PATIENT : profile
    PERSON ||--o| DOCTOR : profile
    APPLICATION_USER ||--o{ HOSPITAL_MEMBERSHIP : joins
    HOSPITAL ||--o{ HOSPITAL_MEMBERSHIP : staff
    HOSPITAL_MEMBERSHIP ||--o{ MEMBERSHIP_ROLE : permissions
~~~

| Таблица | Основни полета и ограничения |
|---|---|
| People | Id, FirstName, MiddleName?, LastName, DateOfBirth?, Gender?, ContactPhone?. Изискванията се валидират при активиране на пациентски/лекарски профил. |
| ApplicationUsers | IdentityUser<Guid>, PersonId UNIQUE; стандартни Identity таблици за вход. Няма един задължителен HospitalId. |
| Hospitals | Id, Name, Slug UNIQUE, City, Address, ContactPhone, ContactEmail, Status (Draft/Active/Suspended). |
| HospitalMemberships | Id, HospitalId, UserId, Status; UNIQUE(HospitalId, UserId). Активиране след потвърдена покана/включване. |
| HospitalMembershipRoles | HospitalId, MembershipId, Role; уникална комбинация. Doctor и HospitalAdministrator са права в тази болница. |
| Patients | Id, PersonId UNIQUE, PlatformPatientNumber UNIQUE, Address?, EmergencyContactName?, EmergencyContactPhone?. |
| Doctors | Id, PersonId UNIQUE, SpecialtyId. |
| Specialties | Id, Code UNIQUE, Name. |
| DoctorDepartments | HospitalId, DoctorId, MembershipId, DepartmentId, WorkEmail?, IsActive; уникално назначение (HospitalId, DoctorId, DepartmentId). |

Patient и PlatformAdministrator могат да са глобални Identity роли. Болничните права не се предоставят от глобална роля Administrator. DoctorDepartments изисква Doctor.PersonId да съответства на PersonId на потребителя от членството и активно лекарско право.

Болничният администратор управлява само своята болница. Лекарят има достъп точно до назначените му отделения; няма наследено право от родителско отделение. Администраторът на платформата няма автоматичен достъп до всички медицински данни.

## 2. Пациенти и медицински данни по болница

~~~mermaid
erDiagram
    HOSPITAL ||--o{ HOSPITAL_PATIENT : registers
    PATIENT ||--o{ HOSPITAL_PATIENT : treated_at
    HOSPITAL_PATIENT ||--o| PATIENT_HEALTH_PROFILE : notes
    HOSPITAL_PATIENT ||--o{ ADMISSION_REQUEST : submits
    ADMISSION_REQUEST ||--o{ ADMISSION_DOCUMENT : supports
~~~

| Таблица | Основни полета и ограничения |
|---|---|
| HospitalPatients | Id, HospitalId, PatientId, LocalPatientNumber?; UNIQUE(HospitalId, PatientId), UNIQUE(HospitalId, LocalPatientNumber) при наличен номер. |
| PatientHealthProfiles | Id, HospitalId, HospitalPatientId UNIQUE, GeneralCondition, Notes?, UpdatedAt, UpdatedByUserId. |
| AdmissionRequests | Id, HospitalId, HospitalPatientId, DepartmentId, PreferredRoomId?, RequestedArrivalAt?, RequestedNights, Reason, Status, SubmittedAt, ReviewedAt?, ReviewedByUserId?, ReviewComment?. |
| AdmissionRequestServices | HospitalId, AdmissionRequestId, HospitalAdditionalServiceId, RequestedQuantity; предпочитания, не окончателна цена. |
| AdmissionDocuments | Id, HospitalId, AdmissionRequestId, UploadedByUserId, OriginalFileName, StorageKey UNIQUE, ContentType, FileSizeBytes, Sha256, ScanStatus, UploadedAt. |

Медицински статус: Submitted, Approved, Rejected, Cancelled, Fulfilled. Желаните нощувки/стая не са одобрение; точните разрешени условия се записват в офертата.

Документите са PDF/JPEG/PNG с проверени тип, размер и съдържание; ScanStatus е Pending/Clean/Rejected. Четенето е през авторизиран endpoint: собствен пациент, болничен администратор или назначен лекар на конкретното отделение.

Медицинските бележки са отделни за всяка болница. Лекарят вижда профил само при текуща заявка за разглеждане или активен престой в свое отделение. Пациентът вижда собствените си изпратени документи, но не вътрешните бележки. Общ акаунт не означава споделено досие между болниците.

## 3. Стаи, легла, цени и услуги

~~~mermaid
erDiagram
    HOSPITAL ||--o{ DEPARTMENT : owns
    DEPARTMENT o|--o{ DEPARTMENT : parent
    DEPARTMENT ||--o{ ROOM : contains
    ROOM_TYPE ||--o{ ROOM : type
    ROOM ||--o{ BED : contains
    ROOM ||--o{ ROOM_RATE : tariffs
    ROOM ||--o{ ROOM_AMENITY : includes
    AMENITY ||--o{ ROOM_AMENITY : describes
~~~

| Таблица | Основни полета и ограничения |
|---|---|
| Departments | Id, HospitalId, Code, Name, Description?, ParentDepartmentId?, IsActive; UNIQUE(HospitalId, Code), без цикли/родител от друга болница. |
| RoomTypes | Id, HospitalId, Code, Name, Description?; UNIQUE(HospitalId, Code). |
| Rooms | Id, HospitalId, DepartmentId, RoomTypeId, RoomNumber, Description?, IsActive; UNIQUE(HospitalId, DepartmentId, RoomNumber). |
| Beds | Id, HospitalId, RoomId, BedNumber, OperationalStatus (InService/OutOfService); UNIQUE(HospitalId, RoomId, BedNumber). |
| Amenities / RoomAmenities | Болничен каталог с Id, HospitalId, Name; свързване HospitalId, RoomId, AmenityId. Без отделно начисляване. |
| RoomRates | Id, HospitalId, RoomId, NightlyRate, Currency, ValidFrom, ValidTo?, Source (HospitalPanel/Api), LastVerifiedAt, SourceUpdatedAt?. Болнична цена за пациент/легло/нощувка, без диапазони по продължителност. Source=Api е запазено за бъдещата интеграция. |
| HospitalAdditionalServices | Id, HospitalId, Code, Name, IsActive; UNIQUE(HospitalId, Code). |
| AdditionalServiceRates | Id, HospitalId, ServiceId, UnitPrice, BillingUnit (PerNight/PerItem/PerStay), Currency, ValidFrom, ValidTo?. Версии на цени. |
| RoomAdditionalServices | HospitalId, RoomId, ServiceId; разрешени допълнения за конкретната стая. |
| HospitalBookingPolicies | Id, HospitalId, Version, DepositType (Fixed/Percent), DepositValue, Currency?, OfferValidityMinutes, HoldMinutes, BalanceDueRule, ValidFrom, ValidTo?. Версионирани условия. |

Болницата поддържа единичната цена на нощувка през панела. При валидността използваме [начало, край); две активни версии за една стая/валута не могат да имат припокриващи се периоди. При създаване на офертата използваме потвърдена действаща болнична цена; редът за настаняване е одобрени нощувки × тази ставка.

Source и времевите полета проследяват произхода и последната проверка на цената. Те не доказват сами по себе си актуалност. При бъдещо получаване чрез API ще използваме външна версия/идентификатор и правила за допустима давност, договорени с болницата. Непотвърдена или изтекла цена не се използва за нов checkout. Обновяване на каталога не променя вече одобрени/приети оферти; промяна на предложената сума изисква нова версия и приемане.

Включените удобства не се добавят като платени услуги. При PerNight количеството е одобреният брой нощувки; при PerItem — одобреният брой; при PerStay — 1. Неподдържана външна тарифа не се превръща приблизително в този модел.

Една стая с друго заето легло е избираема, ако има свободно легло. Публично се показва брой, без данни за настанените пациенти. Текущата наличност не гарантира бъдеща наличност по прогнозна дата на изписване.

## 4. Одобрени оферти, резервации и престои

~~~mermaid
erDiagram
    ADMISSION_REQUEST ||--o{ ADMISSION_OFFER : versions
    ADMISSION_OFFER ||--|{ ADMISSION_OFFER_LINE : prices
    ADMISSION_OFFER ||--o| ADMISSION_BOOKING : accepted
    ADMISSION_BOOKING ||--o{ BED_ALLOCATION : allocated
    BED ||--o{ BED_ALLOCATION : history
    ADMISSION_BOOKING ||--o| HOSPITAL_STAY : admitted
    HOSPITAL_STAY o|--o{ BED_ALLOCATION : occupancy
~~~

| Таблица | Основни полета и ограничения |
|---|---|
| AdmissionOffers | Id, HospitalId, AdmissionRequestId, Version, RoomId, ApprovedNights, ExpectedArrivalAt?, Currency, TotalAmount, DepositAmount, PolicyId, PolicySnapshot, PaymentAccountId?, ExpiresAt, Status, ApprovedByUserId?, ApprovedAt?, AcceptedAt?. |
| AdmissionOfferLines | Id, HospitalId, AdmissionOfferId, LineType (Accommodation/AdditionalService), RoomRateId? / AdditionalServiceRateId?, DescriptionSnapshot, BillingUnitSnapshot, Quantity, UnitPrice, LineTotal. |
| AdmissionBookings | Id, HospitalId, AdmissionRequestId, AdmissionOfferId UNIQUE, PaymentChoice (Full/Deposit/None), Status, HoldExpiresAt?, BalanceDueAt?, CreatedAt. |
| BedAllocations | Id, HospitalId, BedId, Origin, AdmissionBookingId?, HospitalStayId?, Status, HeldAt?, HoldExpiresAt?, ReservedAt?, OccupiedAt?, ReleasedAt?, AssignedByUserId?. |
| HospitalStays | Id, HospitalId, AdmissionBookingId UNIQUE, AdmissionRequestId UNIQUE, DepartmentId, AdmittedAt, ExpectedDischargeAt?, DischargedAt?, Status, AdmittedByUserId, DischargedByUserId?. |

Offer.Status: Draft/Approved/Accepted/Superseded/Expired/Cancelled. UNIQUE(HospitalId, AdmissionRequestId, Version); най-много една текуща Approved/Accepted оферта за заявка. Публикуваните финансови/медицински условия са неизменни; жизненият статус може да се променя. Промени изискват нова версия и ново приемане.

Офертата има точно един Accommodation ред и незадължителни AdditionalService редове. TotalAmount е сумата на LineTotal; депозитът се изчислява по версията на политиката, закръглен до 2 знака. 0 <= DepositAmount <= TotalAmount. При платено настаняване минималното капаро е положително; нулев общ размер допуска None. При депозит, равен на общата сума, няма остатък.

Цената, мерните единици и получателят са записани в приетата оферта. Не пазим втори независимо изменяем TotalAmount в резервацията. Expired е краен статус за неприета оферта; приета оферта е историческият договорен запис и не изтича заради първоначалния срок.

Booking.Status: Holding/Confirmed/Admitted/Completed/Expired/Cancelled/PaymentException. Една текуща резервация за заявка. Повторен checkout след изтекла резервация използва нова валидна оферта. Платените резервации не изтичат по таймера на Holding.

Allocation.Status: Holding/Reserved/Occupied/Completed/Released/Expired. Origin е OnlineBooking или StaffRecordedExternal. OnlineBooking изисква AdmissionBookingId; при Occupied изисква и HospitalStayId. StaffRecordedExternal няма онлайн резервация или фиктивно плащане: персоналът записва реалната заетост извън сайта, със задължителен автор, OccupiedAt и последващо ReleasedAt. Тези записи участват в същите ограничения за заетост. Подробното досие за външния прием е извън v1.

Свободно легло = InService в активна стая без Holding/Reserved/Occupied разпределение. Не се задава ръчно отделен брояч „свободни“. Легло в употреба не се премахва или премества в друга стая.

HospitalStay.Status: Active/Discharged. Настаняването по сайта изисква потвърдена резервация. При смяна на легло в същата стая приключваме старото разпределение и създаваме ново в една транзакция. Промяна на стая/отделение след прием и финансово уреждане са отделно разширение.

## 5. Плащания и автоматична обработка

~~~mermaid
erDiagram
    HOSPITAL ||--o{ HOSPITAL_PAYMENT_ACCOUNT : recipient
    HOSPITAL_PAYMENT_ACCOUNT ||--o{ PAYMENT : processes
    ADMISSION_BOOKING ||--o{ PAYMENT : attempts
    PAYMENT ||--o{ PAYMENT_REFUND : refunds
    PAYMENT o|--o{ PAYMENT_WEBHOOK_EVENT : events
~~~

| Таблица | Основни полета и ограничения |
|---|---|
| HospitalPaymentAccounts | Id, HospitalId, Provider, ProviderAccountId, Status; потвърден получател. Тайните са извън таблицата/репото. |
| Payments | Id, HospitalId, AdmissionBookingId, PaymentAccountId, Purpose (Full/Deposit/Balance), Amount, Currency, ProviderPaymentId?, IdempotencyKey UNIQUE, Status, CreatedAt, CompletedAt?. |
| PaymentWebhookEvents | Id, HospitalId, PaymentAccountId, ProviderEventId, PaymentId?, EventType, ReceivedAt, ProcessedAt?, ProcessingStatus; UNIQUE(PaymentAccountId, ProviderEventId). |
| PaymentRefunds | Id, HospitalId, PaymentId, Amount, Reason, ProviderRefundId?, IdempotencyKey UNIQUE, Status, CreatedAt, CompletedAt?. |

Payment.Status: Created/Pending/Succeeded/Failed/Cancelled. Refund.Status: Pending/Succeeded/Failed. Външните идентификатори са уникални в обхвата на платежния акаунт според гаранциите на доставчика. Непознат/непроверен webhook получател не се свързва по HospitalId от заявката на изпращача.

Платено нето = успешни плащания − успешни възстановявания. Остатък = max(0, сумата по приетата оферта − платено нето). Надплатеното се отчита отделно за възстановяване. Сумата на refund операциите не надвишава полученото по плащането.

Няма плащане преди одобрена оферта и задържане. Full не създава и Deposit. Deposit позволява отделно Balance. Неуспешното първоначално плащане не създава нов дълг; неуспешно Balance не заличава вече договорен остатък. Най-много една неприключила платежна операция за резервация. Превключване Full/Deposit чака изясняване на предходната операция.

Проверяваме подпис, акаунт/получател, сума, валута и външен идентификатор. Браузърно връщане и ръчно променен статус не са доказателство за плащане. Не съхраняваме номера на карти/CVC.

## 6. Поток на прием

~~~mermaid
flowchart TD
    A["Болница, отделение, предпочитания и документ"] --> B{"Решение на персонала"}
    B -->|Отказ| C["Причина за отказ"]
    B -->|Одобрение| D["Конкретна оферта: стая, нощувки и услуги"]
    D --> E["Приемане и повторна проверка на наличността"]
    E --> F{"Има свободно легло"}
    F -->|Не| G["Нов избор и нова оферта"]
    F -->|Да| H["Временно задържане"]
    H --> I["Пълна сума или капаро"]
    I --> J["Проверено успешно плащане"]
    J --> K["Потвърдена резервация"]
    K --> L["Физически прием и престой"]
    K --> M["Остатък само при капаро"]
    L --> N["Изписване и освобождаване"]
~~~

Схемата показва платения успешен път. При нулево доплащане задържането се потвърждава без платежна операция. При окончателен неуспех пациентът може да повтори или откаже според срока на резервацията.

## 7. Часове и проследимост

| Таблица | Основни полета и ограничения |
|---|---|
| AppointmentSlots | Id, HospitalId, DoctorId, DepartmentId, StartsAt, EndsAt, Status (Open/Blocked). Без застъпващи се отворени слотове на един лекар, включително между болници. |
| Appointments | Id, HospitalId, AppointmentSlotId, HospitalPatientId, Status (Scheduled/Cancelled/Completed/NoShow), CreatedAt, CancellationReason?. Един неотменен запис за слот. |
| AuditEvents | Id, HospitalId?, ActorUserId?, Action, EntityType, EntityId, OccurredAt, CorrelationId?, Outcome. NULL болница само за изрично глобални събития. |

Слот с записване не се премества чрез редакция на времето. Пациентски конфликт се проверява по общия PatientId под заключване, без разкриване на записванията в друга болница. Одитът пази автор/действие, но не копира медицински текст или платежни тайни.

## 8. Изолация, ограничения и конкуренция

- Болничните обекти имат UNIQUE(HospitalId, Id) за съставните FK. Например Rooms(HospitalId, DepartmentId) → Departments(HospitalId, Id). Същото се прилага между заявка, пациентска връзка, оферта, стая, легло, престой и плащане.
- Роля се проверява за конкретното членство/ресурс, както при четене, така и при запис. HospitalId идва от проверен сървърен контекст. EF query filters са допълнителна защита; пациентските списъци между болници имат задължителна проверка за собственост.
- PostgreSQL RLS е планирана защита за болничните данни; конкретните политики/DB роли се проектират и тестват в етапа за достъп преди реални данни. Кеш, файлове и фонови задачи следват същия обхват.
- Частичен UNIQUE(BedId) при Holding/Reserved/Occupied важи и за външна заетост. Аналогично UNIQUE(AdmissionBookingId) при активна онлайн алокация и UNIQUE(HospitalStayId) за Occupied.
- Частичен UNIQUE(AdmissionRequestId) за текущи Holding/Confirmed/Admitted/PaymentException резервации. Завършена заявка не позволява нов престой.
- GiST EXCLUDE за BedId и реален интервал [OccupiedAt, ReleasedAt) при OccupiedAt != NULL; незатвореният интервал е безкраен. btree_gist комбинира UUID равенство с интервали.
- EXCLUDE за активните слотове на лекар и за припокриващи се периоди на цените за една стая/валута; частичен UNIQUE за неотменено записване на слот.
- CHECK: положителни нощувки/количества/платежни суми, неотрицателни цени, коректни времеви интервали и депозит до общата сума. Допустимите статуси/Origin и задължителните полета се ограничават изрично.
- Общите суми, допустимите услуги, авторът на одобрение, еднаквият пациент/стая по веригата и един активен онлайн престой на пациент изискват проверки в услугите под заключване плюс подходящи DB ограничения/constraint triggers. Междутaбличните правила не са обикновени CHECK.
- Индексираме FK и списъците: (HospitalId, DepartmentId, Status, SubmittedAt) за заявки; (HospitalId, AdmissionBookingId, Status) за плащания; (HospitalId, BedId, Status) за разпределения.
- Използвани исторически обекти не се изтриват каскадно. Болница/членство/каталог се деактивират; историческите записи остават.

При checkout кратка транзакция заключва заявката/офертата и леглото, създава резервацията и Holding. Мрежовото повикване към платежния доставчик е след commit, с idempotency ключ. Конкуриращи се операции използват еднакъв ред на заключване.

Webhook обработката заключва операцията/резервацията, дедуплицира събитието и не връща Succeeded към Pending. Фоновият процес за изтичане първо изяснява/отменя операцията при доставчика; timeout не е доказан неуспех. Закъснял успех след освобождаване се записва като реално получена сума и PaymentException с техническо възстановяване, без отнемане на друго резервирано легло.

Checkout срокът важи само за Holding. Прието капаро или пълно плащане преминава към Reserved. Частичните индекси са по статус; не използват now(). Неуспешното възстановяване остава за повторни опити и видим сигнал.

## 9. Следващи стъпки и ограничения

След преглед на схемата реализацията започва с модели в Data/Models, отделни конфигурации в Data/Configurations и миграции.

Външна синхронизация не е част от първата миграция; бъдещи HospitalIntegrationConnection и ExternalEntityMapping са в [плана](development-plan.md#интеграции-с-болничните-системи). При такава интеграция е необходимо и външно потвърждение на задържането, ако другата система управлява леглата.

Преди реални плащания се уточняват доставчик/получател, срокове, отказ/неявяване, удължаване и ранно изписване. Ценовото проучване не задава задължителни тарифи на партньорите.

Проверки при реализация: две болници с общ пациент; лекар с две назначения; чужд документ по директен URL; смесени HospitalId; външен прием в последното легло; споделена стая; включена срещу платена услуга; повторен webhook; закъснял успех; Deposit + Balance; промяна на каталожна цена след приета оферта.
