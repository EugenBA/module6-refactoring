# Модуль 6 - рефакториг кода

## 1. Репозиторий
 - исходный код - https://github.com/EugenBA/module6-refactoring/tree/main
 - код после рефакторинга: https://github.com/EugenBA/module6-refactoring/tree/dev
 
## 2. Изменения
### 1 Использование лишних вызовов clone() вместо ссылок.
Трейт Parser переписан с использованием ссылок
```rust
    fn parse<'a>(&self, input: &'a str) -> Result<(&'a str, Self::Dest), ()>;
```
В коде убраны лишние вызовы clone(), в тестах убрано into()

### 2 Использование Rc<RefCell<T>>, там где можно обойтись ссылками.
Код переделан - убраны использование Rc<RefCell<T>>
```rust
struct LogIterator <T: Read + Debug + 'static>{
    lines: std::iter::Filter<Lines<BufReader<T>>,fn(&Result<String, Error>)->bool>,
}
```
### 3 Использование циклов вместо итераторов.
Исправлен цикл на итератор
```rust
pub fn read_log<R: Read + Debug + 'static>(input: R, mode: ReadMode, request_ids: Vec<u32>) -> Vec<LogLine> {
    // подсказка: можно обойтись итераторами
    LogIterator::new(input)
        .filter(|log| {...
```

### 4 Использование unsafe без необходимости.
Метод переписан без использования unsafe
```rust
 Self{
            lines: BufReader::with_capacity(4096, r)
                       .lines()
                       .filter(
                           |line_res|
                           !line_res.as_ref().ok()
                               .map(|line| line.trim().is_empty())
                               .unwrap_or(false)
                        ),
        }
```

### 5 Присутствие singleton, без которого можно обойтись.
Убран синглтон с использованием публиченого метода
```rust
/// Парсер строки логов
pub fn parse(input: &str) -> Result<(&str, LogLine), ()> {
    <LogLine as Parsable>::parser().parse(input)
}
```

### 6 Использование излишней валидации вместо tight-типов.
Использован тип NonZeroU32
```rust
let value = u32::from_str_radix(&remaining[..end_idx], if is_hex { 16 } else { 10 })
                            .ok()
                            .and_then(NonZeroU32::new)
                            .ok_or(())?
                            .get();
```

### 7 Использование вместо дженериков дублирования кода для параметров разных типов.
Метод just_parse_xxx для разных типов переписан с использованием дженериков
```rust
pub fn just_parse<T: Parsable>(input: &str) -> Result<(&str, T), ()> {
    T::parser().parse(input)
}
```
### 8 Использование трейт-объектов в местах, где можно обойтись дженериками.
Код переделан с использованием дженериков
```rust
struct LogIterator <T: Read + Debug + 'static>{
    lines: std::iter::Filter<Lines<BufReader<T>>,fn(&Result<String, Error>)->bool>,
}
impl<T: Read + Debug + 'static> LogIterator<T> {
    fn new(r: T) -> Self...
```
### 9 Использование последовательности if вместо одного match.
Исправлена цепочка if на match
```rust
let mode_matches = match mode {
                ReadMode::All => true,
                ReadMode::Errors => matches!(
                    log.kind,
                    LogKind::System(SystemLogKind::Error(_)) | LogKind::App(AppLogKind::Error(_))
                ),
                ReadMode::Exchanges => matches!(
                    log.kind,
                    LogKind::App(AppLogKind::Journal(
                        AppLogJournalKind::BuyAsset(_)
                            | AppLogJournalKind::SellAsset(_)
                            | AppLogJournalKind::CreateUser { .. }
                            | AppLogJournalKind::RegisterAsset { .. }
                            | AppLogJournalKind::DepositCash(_)
                            | AppLogJournalKind::WithdrawCash(_)
                    ))
                ),
            };
```
Так же if заменен на ленивый вызов парсеров
```rust
 self.parser.0.parse(input)
            .or_else(|_| self.parser.1.parse(input))
            .or_else(|_| self.parser.2.parse(input))
```
### 10 Присутствие Enum, один из вариантов которого занимает несколько килобайт стека.
Использовано выделение в куче для параметра enum
```rust
#[derive(Debug, Clone, PartialEq)]
pub enum AppLogTraceKind {
    Connect(Box<AuthData>),
```
Тип AuthData большого размера
```rust
const AUTHDATA_SIZE: usize = 1024;
#[derive(Debug,Clone,PartialEq)]
pub struct AuthData(Box<[u8;AUTHDATA_SIZE]>);
```
### 11 Использование паники вместо возврата ошибки.
Использован enum для типа парсинга, что так же сделало не нужной ветку с паникой
```rust
let mode_matches = match mode {
                ReadMode::All => true,
                ReadMode::Errors => matches!(
                    log.kind,
                    LogKind::System(SystemLogKind::Error(_)) | LogKind::App(AppLogKind::Error(_))
                ),
                ReadMode::Exchanges => matches!(
                    log.kind,
                    LogKind::App(AppLogKind::Journal(
                        AppLogJournalKind::BuyAsset(_)
                            | AppLogJournalKind::SellAsset(_)
                            | AppLogJournalKind::CreateUser { .. }
                            | AppLogJournalKind::RegisterAsset { .. }
                            | AppLogJournalKind::DepositCash(_)
                            | AppLogJournalKind::WithdrawCash(_)
                    ))
                ),
            };
```
### 12 Логические ошибки
Найдена логическая ошибка
```rust
map(
    preceded(strip_whitespace(tag("WithdrawCash")), UserCash::parser()),
    |user_cash| AppLogJournalKind::DepositCash(user_cash) // 
),
```
Исправлено на WithdrawCash

## 3. Тесты
Тесты проходят успешно
```text

running 16 tests
test parse::test::test_asset_dsc ... ok
test parse::test::test_delimited ... ok
test parse::test::test_backet ... ok
test parse::test::test_do_unquote_non_escaped ... ok
test parse::test::test_i32 ... ok
test parse::test::test_key_value ... ok
test parse::test::test_list ... ok
test parse::test::test_quote ... ok
test parse::test::test_authdata ... ok
test parse::test::test_quoted_tag ... ok
test parse::test::test_strip_whitespace ... ok
test parse::test::test_tag ... ok
test parse::test::test_u32 ... ok
test parse::test::test_unquote ... ok
test parse::test::test_log_kind ... ok
test test::test_all ... ok

test result: ok. 16 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/cli-4f4a3e7ba7357f9d)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests analysis

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

```