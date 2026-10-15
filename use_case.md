```mermaid
usecase-beta
direction LR

actor User("Пользователь")
actor Officer("Сотрудник УМВД")

systemBoundary portal["Портал миграционного учета"]
  Register("Зарегистрироваться")
  Confirm("Подтвердить телефон")
  Login("Войти в систему")
  SendNotice("Отправить уведомление о постановке на учёт")
  AttachDocs("Прикрепить копии документов")
  TrackStatus("Отслеживать статус уведомления")
  GetDoc("Получить документ о постановке на учет")
  Review("Рассмотреть уведомление")
  Reject("Отказать в постановке на учёт")
  IssueDoc("Загрузить документ о постановке на учет")
  Citizenship("Определить сроки по гражданству")
end

User -- Register
User -- Login
User -- SendNotice
User -- TrackStatus
Officer -- Login
Officer -- Review

Citizenship ..> : include SendNotice
AttachDocs ..> : include SendNotice
Register ..> : include Confirm
Review ..> : include IssueDoc

TrackStatus ..> : extend GetDoc
Reject ..> : extend Review
```
