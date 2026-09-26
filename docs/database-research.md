# Проучване на болнични информационни системи

Проверено на 26.09.2026. Разгледани са официална документация и публичен изходен код. Това не е достъп до производствените бази на конкретни български болници. Изводите по-долу са наша адаптация за HospitalApp, а не копиране на цял чужд продукт.

## Сравнение и решения

| Система / първичен източник | Наблюдаван модел | Приложение при нас |
|---|---|---|
| [OpenMRS — data model](https://github.com/openmrs/openmrs-book-developer-manual/blob/main/Technology/dataModel.md) | Person отделя общите лични данни от Patient и User; Encounter описва клиничен контакт. | People + ApplicationUsers + Patients + Doctors; не дублираме име и дата на раждане при няколко роли. |
| [OpenMRS — visits](https://guide.openmrs.org/configuration/configuring-visits/) | Visit групира клинични контакти. | Отделяме HospitalStay от предварителната заявка; не въвеждаме целия клиничен модел във v1. |
| [OpenMRS/Bahmni — bed management](https://github.com/openmrs/openmrs-module-bedmanagement/blob/master/api/src/main/resources/liquibase.xml) | Миграциите съдържат bed и bed_patient_assignment_map с пациент, легло и начало/край на разпределението. | Beds и история BedAllocations; не заменяме историята с един BedId в профила на пациента. |
| [Frappe Health / Marley — Inpatient Record](https://github.com/earthians/marley/blob/develop/healthcare/healthcare/doctype/inpatient_record/inpatient_record.json) и [Inpatient Occupancy](https://github.com/earthians/marley/blob/develop/healthcare/healthcare/doctype/inpatient_occupancy/inpatient_occupancy.json) | Inpatient Record има списък inpatient_occupancies; Occupancy съдържа service_unit, check_in и check_out. | Престоят и разпределението на легло са отделни записи, с възможност за история. |
| [Marley — Healthcare Service Unit](https://github.com/earthians/marley/blob/develop/healthcare/healthcare/doctype/healthcare_service_unit/healthcare_service_unit.json) | Йерархия с parent_healthcare_service_unit и тип единица. | Запазваме Department.ParentDepartmentId и ясни отделни Room/Bed таблици, подходящи за по-малкия ни обхват. |
| [Open Hospital — Admission entity](https://github.com/informatici/openhospital-core/blob/develop/src/main/java/org/isf/admission/model/Admission.java) | Отделен admission запис, свързан с Patient и Ward, с данни за прием и изписване. | HospitalStay има собствен жизнен цикъл и връзка към одобреното отделение. |
| [ERPNext — Payment Entry](https://docs.frappe.io/erpnext/payment-entry) | Отделни платежни записи, включително аванси и частични плащания. | Payments и неизменна оферта; медицинският статус не съдържа DepositPaid/PaidInFull. |

OpenMRS и Bahmni са свързана екосистема, не две независими потвърждения. Публичното хранилище frappe/health препраща към earthians/marley; използван е наличният там код. Линковете към develop/master са подвижни, затова датата на проверката е посочена.

## Приложение в HospitalApp

Разделянето AdmissionRequest → AdmissionOffer → AdmissionBooking → HospitalStay съчетава медицинското одобрение, конкретните финансови условия, избора на свободно място и последващия физически прием. Именно онлайн изборът Full/Deposit и ограниченията по болница и отделение налагат допълнителните обекти. Не твърдим, че всички проучени системи имат същия checkout процес.

Единната BedAllocation таблица обхваща и временно задържане, и резервация, и реална заетост. Това е наше решение за общо ограничение срещу двойно заемане. Наличността се изчислява по легла. Историческите записи остават след изписване.

Цената на нощувка се запазва в офертата. Платената сума и остатъкът се извеждат от операциите, а не се копират като променящ се баланс в медицинския престой. Отделяме техническите възстановявания при платежен конфликт от бъдещата политика за доброволни откази.

## Технически основания

- [PostgreSQL — range constraints](https://www.postgresql.org/docs/current/rangetypes.html#RANGETYPES-CONSTRAINT): exclusion constraints за незастъпващи се интервали и btree_gist за комбиниране с идентификатор на ресурс.
- [PostgreSQL — constraints](https://www.postgresql.org/docs/current/ddl-constraints.html): уникалност, външни ключове и ограничения; свързващите FK колони изискват обмислени индекси.
- [Stripe — webhooks](https://docs.stripe.com/webhooks): проверка на подписи, повторни и неподредени събития. Използвано като конкретен пример за интеграционни изисквания; платежен доставчик още не е избран.

## Какво не пренасяме във v1

Общият пациентски профил се свързва чрез HospitalPatient с отделни медицински бележки във всяка болница. HospitalMembership определя правата на персонала. Това е наша адаптация, а не твърдение, че разгледаните клинични системи са платформи за общ пациентски каталог между независими болници.

Началният източник на наличност е болничният панел. Автоматичната синхронизация е бъдещ етап в [плана за разработка](development-plan.md#интеграции-с-болничните-системи).

Не копираме универсалния клиничен речник на OpenMRS, лабораторни и аптечни модули, пълна счетоводна система или всички FHIR ресурси. Публичните модели дават структурни насоки; конкретните таблици остават съобразени с договорения малък обхват.

Проектът на базата е описан в [схемата и диаграмите](database-design.md). Публичните български ценови примери са в [отделното ценово проучване](pricing-research-bg.md). То служи за справка. Цените в приложението идват от болниците, а обновяване чрез техни API е бъдещ етап.
