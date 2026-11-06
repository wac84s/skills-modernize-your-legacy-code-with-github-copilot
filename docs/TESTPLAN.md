# Test Plan: COBOL Account Management System

## 1. Purpose and scope

This test plan documents the business logic of the legacy COBOL Account Management System. It is intended to:

- Let business stakeholders confirm which behaviours are correct and which are defects.
- Serve as the basis for unit and integration tests in the Node.js application that replaces the COBOL system.

The plan covers every business rule in the three COBOL programs:

| Source file | Program | Responsibility |
|---|---|---|
| [main.cob](/workspaces/skills-modernize-your-legacy-code-with-github-copilot/src/cobol/main.cob) | `MainProgram` | Menu loop, user choice handling, exit |
| [operations.cob](/workspaces/skills-modernize-your-legacy-code-with-github-copilot/src/cobol/operations.cob) | `Operations` | View balance, credit, debit business rules |
| [data.cob](/workspaces/skills-modernize-your-legacy-code-with-github-copilot/src/cobol/data.cob) | `DataProgram` | In-memory storage of the balance (READ / WRITE) |

## 2. Business rules (as implemented)

| Rule | Description |
|---|---|
| BR-01 | The account opens with a balance of **1000.00** each time the program starts. |
| BR-02 | The balance is held in memory only. It is not saved between program runs. |
| BR-03 | The balance is stored and displayed with 6 integer digits and 2 decimal places (`PIC 9(6)V99`), for example `001000.00`. |
| BR-04 | A credit adds the entered amount to the balance. |
| BR-05 | A debit subtracts the entered amount only if the balance is greater than or equal to the amount. Otherwise the debit is refused. |
| BR-06 | The balance cannot go negative through a debit. |
| BR-07 | Menu choices are 1 (view), 2 (credit), 3 (debit), and 4 (exit). Any other value shows an error and redisplays the menu. |

## 3. Test environment and how to run

- **Compiler:** GnuCOBOL 3.1.2.0 (`cobc`). Results below were observed with this version. Behaviour around invalid input can differ between COBOL compilers.
- **Build:** `cobc -x src/cobol/main.cob src/cobol/operations.cob src/cobol/data.cob -o accountsystem`
- **Run:** `./accountsystem`
- Each test starts with a fresh run of the program so the balance is 1000.00 (unless a pre-condition says otherwise).
- Inputs in the steps are typed at the prompts, in order.

## 4. Status key

- **Pass:** Actual result matches the expected result.
- **Fail:** Actual result differs from the expected result. The Comments column explains the difference and whether a stakeholder decision is needed.

Expected results describe the behaviour the business should have. Where the COBOL code gives a different result, the case is marked **Fail** so that stakeholders can confirm the defect.

## 5. Test cases

### 5.1 Menu and navigation

| Test Case ID | Test Case Description | Pre-conditions | Test Steps | Expected Result | Actual Result | Status (Pass/Fail) | Comments |
|---|---|---|---|---|---|---|---|
| MNU-01 | Menu is displayed when the program starts | Program is built | 1. Run `./accountsystem` | Header "Account Management System", options 1. View Balance, 2. Credit Account, 3. Debit Account, 4. Exit, and prompt "Enter your choice (1-4):" are shown | Shown as expected | Pass | |
| MNU-02 | Choice 4 exits the program | Program is running | 1. Enter `4` | "Exiting the program. Goodbye!" is shown and the program ends | "Exiting the program. Goodbye!" shown, program ended | Pass | |
| MNU-03 | Choice 0 is rejected | Program is running | 1. Enter `0`<br>2. Enter `4` | "Invalid choice, please select 1-4." is shown, menu is redisplayed, then program exits on `4` | Error message shown, menu redisplayed, exited on `4` | Pass | |
| MNU-04 | Choice 5 is rejected | Program is running | 1. Enter `5`<br>2. Enter `4` | "Invalid choice, please select 1-4." is shown, menu is redisplayed | Error message shown, menu redisplayed | Pass | |
| MNU-05 | Choice 9 is rejected | Program is running | 1. Enter `9`<br>2. Enter `4` | "Invalid choice, please select 1-4." is shown, menu is redisplayed | Error message shown, menu redisplayed | Pass | |
| MNU-06 | Non-numeric choice is rejected | Program is running | 1. Enter `A`<br>2. Enter `4` | "Invalid choice, please select 1-4." is shown, menu is redisplayed | Error message shown, menu redisplayed | Pass | |
| MNU-07 | Blank choice is rejected | Program is running | 1. Press Enter with no value<br>2. Enter `4` | "Invalid choice, please select 1-4." is shown, menu is redisplayed | Error message shown, menu redisplayed | Pass | |
| MNU-08 | Multi-digit choice `10` is rejected | Program is running | 1. Enter `10`<br>2. Enter `4` | "Invalid choice, please select 1-4." is shown and the balance is not displayed | "Current balance: 001000.00" shown. Only the first digit (`1`) was used | Fail | The menu field holds one digit, so `10` is read as `1`. Should be rejected as invalid. Defect. |
| MNU-09 | Multi-digit choice `12` is rejected | Program is running | 1. Enter `12`<br>2. Enter `4` | "Invalid choice, please select 1-4." is shown | "Current balance: 001000.00" shown. Read as `1` | Fail | Same cause as MNU-08. Defect. Important: a user intending `12` gets a balance view, not an error. |
| MNU-10 | Multi-digit choice `20` is rejected | Program is running | 1. Enter `20`<br>2. Enter `500`<br>3. Enter `1`<br>4. Enter `4` | "Invalid choice, please select 1-4." is shown and no credit is made | Read as `2`. Credit prompt shown and 500.00 credited (balance 1500.00) | Fail | Same cause as MNU-08. A money movement happens from an invalid menu entry. High severity defect. |
| MNU-11 | Operations can be repeated in one session | Program is running | 1. Enter `1`<br>2. Enter `1`<br>3. Enter `1`<br>4. Enter `4` | Balance is shown three times (001000.00 each time), then program exits | Balance shown three times as 001000.00, then exited | Pass | |
| MNU-12 | Program handles end of input (closed stdin) gracefully | Program is running with input piped from a file or closed stdin | 1. Pipe input `1` with no further input and no `4` | Program exits, or reports an error, instead of running forever | Menu repeats endlessly and the program never ends (stopped by a 3-second timeout during testing) | Fail | ACCEPT at end of input returns no value, which is treated as an invalid choice in a loop. Defect for scripted or automated use. Expected behaviour is an exit. |

### 5.2 Balance viewing and formatting

| Test Case ID | Test Case Description | Pre-conditions | Test Steps | Expected Result | Actual Result | Status (Pass/Fail) | Comments |
|---|---|---|---|---|---|---|---|
| BAL-01 | Opening balance is 1000.00 on a fresh start | Fresh program start (BR-01) | 1. Run the program<br>2. Enter `1`<br>3. Enter `4` | "Current balance: 001000.00" | "Current balance: 001000.00" | Pass | Confirms BR-01 and BR-03 display format. |
| BAL-02 | Viewing the balance does not change it | Balance is 1000.00 | 1. Enter `1` three times<br>2. Enter `4` | Balance stays 001000.00 on every view | 001000.00 on every view | Pass | |
| BAL-03 | View shows the balance after a credit in the same session | Fresh start | 1. Enter `2`<br>2. Enter `250.50`<br>3. Enter `1`<br>4. Enter `4` | "Current balance: 001250.50" | "Current balance: 001250.50" | Pass | |
| BAL-04 | Balance display always uses 6 integer digits and 2 decimals | Any balance | 1. Enter `2`, then `0` (zero credit)<br>2. Enter `1`<br>3. Enter `4` | Balance is shown as `001000.00` with leading zeros and two decimals | "001000.00" | Pass | Format is fixed width. Downstream systems and reports must expect this. |

### 5.3 Credit account

| Test Case ID | Test Case Description | Pre-conditions | Test Steps | Expected Result | Actual Result | Status (Pass/Fail) | Comments |
|---|---|---|---|---|---|---|---|
| CRD-01 | Credit a decimal amount | Balance 1000.00 | 1. Enter `2`<br>2. Enter `250.50`<br>3. Enter `4` | "Amount credited. New balance: 001250.50" | "Amount credited. New balance: 001250.50" | Pass | Core credit rule (BR-04). |
| CRD-02 | Credit a whole-number amount | Balance 1000.00 | 1. Enter `2`<br>2. Enter `100`<br>3. Enter `4` | "New balance: 001100.00" | "New balance: 001100.00" | Pass | |
| CRD-03 | Credit an amount with no whole part | Balance 1000.00 | 1. Enter `2`<br>2. Enter `.5`<br>3. Enter `4` | "New balance: 001000.50" | "New balance: 001000.50" | Pass | |
| CRD-04 | Credit an amount of zero | Balance 1000.00 | 1. Enter `2`<br>2. Enter `0`<br>3. Enter `4` | Balance stays 001000.00. Confirmation is shown | "New balance: 001000.00" | Pass | Zero credit is accepted. Stakeholders may want to reject zero amounts (see Section 6). |
| CRD-05 | Credit 1000.00 on a 1000.00 balance | Balance 1000.00 | 1. Enter `2`<br>2. Enter `1000`<br>3. Enter `4` | "New balance: 002000.00" | "New balance: 002000.00" | Pass | |
| CRD-06 | Credit to the largest balance the system can hold | Balance 1000.00 | 1. Enter `2`<br>2. Enter `998999`<br>3. Enter `4` | "New balance: 999999.00" | "New balance: 999999.00" | Pass | Upper limit of the balance field is 999999.99. |
| CRD-07 | Credit that would exceed the balance limit is rejected | Balance 1000.00 | 1. Enter `2`<br>2. Enter `999999.99`<br>3. Enter `4` | Credit is rejected with an error message. Balance stays 001000.00 | "New balance: 000999.90" | Fail | Result is silently wrong. The overflow is not detected, so the balance is corrupted. Defect, high severity. |
| CRD-08 | Credit of 999000 that overflows is rejected | Balance 1000.00 | 1. Enter `2`<br>2. Enter `999000`<br>3. Enter `4` | Credit is rejected with an error message. Balance stays 001000.00 | "New balance: 000000.00" | Fail | Total of 1000000.00 wraps to 0.00 and money is lost. Defect, high severity. |
| CRD-09 | Credit amount with more than 6 whole digits is rejected | Balance 1000.00 | 1. Enter `2`<br>2. Enter `1000000`<br>3. Enter `4` | Credit is rejected with an error message. Balance stays 001000.00 | "Amount credited. New balance: 001000.00" | Fail | Amount is silently cut to `000000` and reported as credited. Defect. |
| CRD-10 | Credit amount with 7 digits is rejected | Balance 1000.00 | 1. Enter `2`<br>2. Enter `1234567`<br>3. Enter `4` | Credit is rejected with an error message. Balance stays 001000.00 | "New balance: 235567.00" | Fail | Amount silently cut to `234567`. Customer is credited the wrong amount. Defect, high severity. |
| CRD-11 | Negative credit amount is rejected | Balance 1000.00 | 1. Enter `2`<br>2. Enter `-50`<br>3. Enter `4` | Credit is rejected with an error message. Balance stays 001000.00 | "New balance: 001050.00" | Fail | The minus sign is dropped, so -50 is treated as +50. A "credit" can add money in the wrong direction. Defect, high severity. |
| CRD-12 | Amount with more than 2 decimal places | Balance 1000.00 | 1. Enter `2`<br>2. Enter `10.567`<br>3. Enter `4` | Amount is either rejected, or rounded to 2 decimal places (10.57) | "New balance: 001010.56" | Fail | Extra decimals are truncated (not rounded), giving 10.56. Stakeholder decision needed on reject vs. round (see Section 6). |
| CRD-13 | Non-numeric credit amount is rejected | Balance 1000.00 | 1. Enter `2`<br>2. Enter `abc`<br>3. Enter `4` | Credit is rejected with an error message. Balance stays 001000.00 | "Amount credited. New balance: 001000.00" | Fail | Text is read as 0 and reported as a successful credit. Defect. |

### 5.4 Debit account

| Test Case ID | Test Case Description | Pre-conditions | Test Steps | Expected Result | Actual Result | Status (Pass/Fail) | Comments |
|---|---|---|---|---|---|---|---|
| DBT-01 | Debit an amount that is covered by the balance | Balance 1000.00 | 1. Enter `3`<br>2. Enter `300`<br>3. Enter `4` | "Amount debited. New balance: 000700.00" | "Amount debited. New balance: 000700.00" | Pass | Core debit rule (BR-05). |
| DBT-02 | Debit the exact balance | Balance 1000.00 | 1. Enter `3`<br>2. Enter `1000`<br>3. Enter `4` | "Amount debited. New balance: 000000.00" | "Amount debited. New balance: 000000.00" | Pass | Boundary: balance equal to amount is allowed. |
| DBT-03 | Debit one cent more than the balance is refused | Balance 1000.00 | 1. Enter `3`<br>2. Enter `1000.01`<br>3. Enter `4` | "Insufficient funds for this debit." Balance stays 001000.00 | "Insufficient funds for this debit." Balance 001000.00 | Pass | Boundary: balance cannot go negative (BR-06). |
| DBT-04 | Debit of zero | Balance 1000.00 | 1. Enter `3`<br>2. Enter `0`<br>3. Enter `4` | Balance stays 001000.00 | "New balance: 001000.00" | Pass | Zero debit is accepted. Stakeholders may want to reject it (see Section 6). |
| DBT-05 | Debit from a balance of zero is refused | Balance is 0.00 (after DBT-02) | 1. Enter `3`<br>2. Enter `1000`<br>3. Enter `3`<br>4. Enter `1`<br>5. Enter `4` | First debit: balance 000000.00. Second debit of 1 is refused with "Insufficient funds for this debit." | First debit: 000000.00. Second debit: "Insufficient funds for this debit." | Pass | |
| DBT-06 | Debit after a credit uses the updated balance | Balance 1000.00 | 1. Enter `2`<br>2. Enter `500`<br>3. Enter `3`<br>4. Enter `1500`<br>5. Enter `4` | Credit gives 1500.00. Debit of 1500 succeeds and balance becomes 000000.00 | "New balance: 001500.00", then "New balance: 000000.00" | Pass | Confirms that credits and debits share the same balance. |
| DBT-07 | Negative debit amount is rejected | Balance 1000.00 | 1. Enter `3`<br>2. Enter `-100`<br>3. Enter `4` | Debit is rejected with an error message. Balance stays 001000.00 | "Amount debited. New balance: 000900.00" | Fail | The minus sign is dropped, so the debit of -100 is taken as 100. Defect, high severity. |
| DBT-08 | Non-numeric debit amount is rejected | Balance 1000.00 | 1. Enter `3`<br>2. Enter `abc`<br>3. Enter `4` | Debit is rejected with an error message. Balance stays 001000.00 | "Amount debited. New balance: 001000.00" | Fail | Text is read as 0 and reported as a successful debit. Defect. |

### 5.5 Data storage (balance persistence)

| Test Case ID | Test Case Description | Pre-conditions | Test Steps | Expected Result | Actual Result | Status (Pass/Fail) | Comments |
|---|---|---|---|---|---|---|---|
| DAT-01 | Balance is reset to 1000.00 on each new program run | A previous run credited 250.50 and ended | 1. Run the program with input `2`, `250.50`, `4`<br>2. Run the program again with input `1`, `4` | Second run shows "Current balance: 001000.00" | Second run: "Current balance: 001000.00" | Pass | Matches BR-01 and BR-02 as implemented. **Stakeholders must confirm.** If balances must survive restarts, this is a missing requirement (see Section 6). |
| DAT-02 | Balance written by a credit is read back by the next operation | Balance 1000.00 | 1. Enter `2`, `250.50`<br>2. Enter `1`<br>3. Enter `3`, `100`<br>4. Enter `1`<br>5. Enter `4` | Views show 001250.50, then 001150.50 | Balance views show 001250.50 and then 001150.50 | Pass | Confirms that the read and write steps in `DataProgram` work. |

## 6. Summary of defects and open questions

### 6.1 Defects found (Status = Fail)

| Severity | Test Case IDs | Defect | Business impact |
|---|---|---|---|
| High | CRD-07, CRD-08 | Credits that exceed the 999999.99 limit wrap around silently. | Balance is corrupted and money is lost. |
| High | CRD-10 | Amounts with more than 6 whole digits are truncated (for example `1234567` becomes `234567`). | Wrong amount is credited. |
| High | CRD-11, DBT-07 | Minus sign is ignored. | A negative credit or debit moves money in the wrong direction. Debit can be used to add money. |
| High | MNU-10 | Multi-digit menu input is read by its first digit (`20` is read as `2`). | A menu entry that is invalid can still cause a credit. |
| Medium | MNU-08, MNU-09 | Multi-digit menu input is read by its first digit (`10` and `12` are read as `1`). | Users see a balance view instead of an error. |
| Medium | CRD-09 | Credit of 1000000 is reported as successful but adds 0. | Customer is told a credit happened when it did not. |
| Medium | CRD-13, DBT-08 | Non-numeric amounts are treated as 0 and reported as successful. | Misleading success messages. |
| Medium | MNU-12 | Program loops forever when input ends. | Program cannot be used in scripts or automated tests. |
| Low | CRD-12 | Extra decimal places are truncated instead of rejected or rounded. | Small amounts lost. Stakeholder decision needed. |

### 6.2 Questions for business stakeholders

1. **Persistence (BR-02, DAT-01):** Must the balance survive a restart? The current system always starts at 1000.00.
2. **Opening balance (BR-01):** Is 1000.00 the correct opening balance for every account, or is it a demo value?
3. **Zero amounts (CRD-04, DBT-04):** Should zero credits and debits be rejected?
4. **Decimal places (CRD-12):** When more than 2 decimals are entered, should the system reject the amount or round it? If rounding, which rule (for example half-up)?
5. **Maximum balance (CRD-06 to CRD-08):** What is the maximum balance? Should credits above it be rejected?
6. **Error messages:** Should invalid amounts show a specific message, such as "Invalid amount"? Only the menu has an error message today.

### 6.3 Scope notes

- Operations other than view, credit, debit, and exit do not exist in the current system. There are no account numbers, customer records, transaction history, or audit trail. These are out of scope for this plan.
- Tests in this plan are black-box tests based on the console program. They are written so that the same cases can be automated against the Node.js replacement.
