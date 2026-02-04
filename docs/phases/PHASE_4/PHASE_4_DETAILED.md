# Phase 4: Allowances & Automation
**Duration: 2 weeks**

---

## Overview

Phase 4 implements automated allowance payments, interest calculations, and transaction reporting. This phase turns PicklesApp into a complete family financial management platform.

---

## Goals

- Automated allowance payments
- Interest calculations
- Transaction reporting

---

## Deliverables

### 1. Backend

- Allowance and Split models
- Allowance payment processor (scheduled jobs)
- Interest calculation engine
- Automated debit system
- CSV export endpoint

### 2. Frontend

- Allowance creation form
  - Amount or formula (age-based)
  - Frequency selection
  - Split configuration
- Split builder UI (drag percentages)
- Interest calculator
- Transaction export UI
- Weekly report preview

### 3. Features

- Weekly, bi-weekly, monthly, semi-monthly allowances
- Age-based formulas (age * 2)
- Split deposits (60/20/20)
- Parent-paid interest (compound)
- Interest caps
- Automated recurring debits
- CSV transaction export

---

## Database Schema

### Allowance Table

```sql
CREATE TABLE allowances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    amount_type VARCHAR(20) NOT NULL CHECK (amount_type IN ('fixed', 'formula')),
    amount DECIMAL(10, 2),  -- For fixed amounts
    formula TEXT,  -- For age-based formulas: "age * 2"
    frequency VARCHAR(20) NOT NULL CHECK (frequency IN ('weekly', 'biweekly', 'semimonthly', 'monthly')),
    day_of_week INTEGER,  -- 0-6 for weekly
    days_of_month INTEGER[],  -- [1, 15] for semi-monthly
    is_active BOOLEAN DEFAULT true,
    next_payment_date DATE,
    last_payment_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_allowances_family ON allowances(family_id);
CREATE INDEX idx_allowances_next_payment ON allowances(next_payment_date) WHERE is_active = true;
```

### Allowance Split Table

```sql
CREATE TABLE allowance_splits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    allowance_id UUID NOT NULL REFERENCES allowances(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    percentage DECIMAL(5, 2) NOT NULL CHECK (percentage >= 0 AND percentage <= 100),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Ensure percentages sum to 100%
CREATE OR REPLACE FUNCTION check_split_percentages()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT SUM(percentage) FROM allowance_splits WHERE allowance_id = NEW.allowance_id) > 100 THEN
        RAISE EXCEPTION 'Split percentages cannot exceed 100%%';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_splits
AFTER INSERT OR UPDATE ON allowance_splits
FOR EACH ROW EXECUTE FUNCTION check_split_percentages();
```

### Interest Configuration Table

```sql
CREATE TABLE interest_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    annual_rate DECIMAL(5, 4) NOT NULL CHECK (annual_rate >= 0 AND annual_rate <= 1),  -- 0.05 = 5%
    compound_frequency VARCHAR(20) NOT NULL DEFAULT 'monthly' CHECK (compound_frequency IN ('daily', 'weekly', 'monthly', 'yearly')),
    cap_amount DECIMAL(10, 2),  -- Maximum balance for interest
    is_active BOOLEAN DEFAULT true,
    last_calculated_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(account_id)
);
```

### Recurring Debit Table

```sql
CREATE TABLE recurring_debits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    amount DECIMAL(10, 2) NOT NULL CHECK (amount > 0),
    frequency VARCHAR(20) NOT NULL CHECK (frequency IN ('weekly', 'biweekly', 'monthly')),
    day_of_week INTEGER,
    day_of_month INTEGER,
    is_active BOOLEAN DEFAULT true,
    next_debit_date DATE,
    last_debit_date DATE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## Allowance Payment Processing

### Payment Processor

```python
from decimal import Decimal
from datetime import date, timedelta
from apscheduler.schedulers.asyncio import AsyncIOScheduler

async def process_allowance_payments():
    """
    Process all due allowances. 
    Runs daily at midnight in family's timezone.
    """
    today = date.today()
    
    # Get all due allowances
    due_allowances = db.query(Allowance).filter(
        Allowance.is_active == True,
        Allowance.next_payment_date <= today
    ).all()
    
    for allowance in due_allowances:
        try:
            await process_single_allowance(allowance)
        except Exception as e:
            logger.error(f"Failed to process allowance {allowance.id}: {e}")
            # Continue processing other allowances

async def process_single_allowance(allowance: Allowance):
    """
    Process a single allowance payment.
    """
    # Calculate amount
    if allowance.amount_type == 'fixed':
        amount = allowance.amount
    else:
        # Formula-based (e.g., age * 2)
        user = get_user(allowance.user_id)
        age = calculate_age(user.birthdate)
        amount = eval_formula(allowance.formula, {'age': age})
    
    # Get splits
    splits = get_allowance_splits(allowance.id)
    
    if not splits:
        # No splits - deposit to primary spending account
        primary_account = get_primary_account(allowance.user_id, allowance.family_id)
        create_transaction(
            account_id=primary_account.id,
            amount=amount,
            transaction_type='ALLOWANCE',
            description=f"Allowance: {allowance.name}"
        )
    else:
        # Process splits
        for split in splits:
            split_amount = (amount * split.percentage / 100).quantize(Decimal('0.01'))
            create_transaction(
                account_id=split.account_id,
                amount=split_amount,
                transaction_type='ALLOWANCE',
                description=f"Allowance: {allowance.name} ({split.percentage}%)"
            )
    
    # Update next payment date
    allowance.last_payment_date = date.today()
    allowance.next_payment_date = calculate_next_payment_date(allowance)
    
    db.commit()

def calculate_next_payment_date(allowance: Allowance) -> date:
    """
    Calculate the next payment date based on frequency.
    """
    current = allowance.next_payment_date
    
    if allowance.frequency == 'weekly':
        return current + timedelta(weeks=1)
    elif allowance.frequency == 'biweekly':
        return current + timedelta(weeks=2)
    elif allowance.frequency == 'monthly':
        return add_months(current, 1)
    elif allowance.frequency == 'semimonthly':
        # 1st and 15th
        if current.day < 15:
            return current.replace(day=15)
        else:
            next_month = add_months(current, 1)
            return next_month.replace(day=1)
```

### Age-Based Formula Evaluation

```python
import re

ALLOWED_FORMULA_VARS = {'age', 'years_in_family'}
ALLOWED_OPERATORS = {'+', '-', '*', '/', '(', ')', '.', ' '}

def eval_formula(formula: str, context: dict) -> Decimal:
    """
    Safely evaluate age-based formula.
    Examples: "age * 2", "age * 1.5 + 5", "(age - 5) * 2"
    """
    # Validate formula contains only allowed tokens
    cleaned = formula
    for var in ALLOWED_FORMULA_VARS:
        cleaned = cleaned.replace(var, '')
    
    for char in cleaned:
        if char not in ALLOWED_OPERATORS and not char.isdigit():
            raise ValueError(f"Invalid formula character: {char}")
    
    # Replace variables with values
    for var, value in context.items():
        formula = formula.replace(var, str(value))
    
    # Evaluate
    result = eval(formula)
    return Decimal(str(result)).quantize(Decimal('0.01'))
```

---

## Interest Calculation Engine

### Interest Calculator

```python
from decimal import Decimal
from math import pow

async def calculate_interest():
    """
    Calculate and apply interest for all accounts with interest configs.
    Runs monthly on the 1st at midnight.
    """
    configs = db.query(InterestConfig).filter(
        InterestConfig.is_active == True
    ).all()
    
    for config in configs:
        try:
            await apply_interest(config)
        except Exception as e:
            logger.error(f"Failed to apply interest for account {config.account_id}: {e}")

async def apply_interest(config: InterestConfig):
    """
    Calculate and apply compound interest to an account.
    """
    account = get_account(config.account_id)
    
    # Apply cap if set
    eligible_balance = account.balance
    if config.cap_amount and account.balance > config.cap_amount:
        eligible_balance = config.cap_amount
    
    # Calculate interest based on compound frequency
    if config.compound_frequency == 'monthly':
        # Monthly compounding
        monthly_rate = config.annual_rate / 12
        interest = eligible_balance * monthly_rate
    elif config.compound_frequency == 'weekly':
        # Weekly compounding (4.33 weeks since last calculation)
        weekly_rate = config.annual_rate / 52
        weeks = 4.33  # Average weeks in a month
        interest = eligible_balance * (pow(1 + float(weekly_rate), weeks) - 1)
    elif config.compound_frequency == 'daily':
        # Daily compounding
        daily_rate = config.annual_rate / 365
        days = 30  # Approximate month
        interest = eligible_balance * (pow(1 + float(daily_rate), days) - 1)
    else:  # yearly
        # Apply 1/12 of annual interest monthly
        interest = eligible_balance * (config.annual_rate / 12)
    
    interest_amount = Decimal(str(interest)).quantize(Decimal('0.01'))
    
    if interest_amount > 0:
        create_transaction(
            account_id=account.id,
            amount=interest_amount,
            transaction_type='INTEREST',
            description=f"Interest ({config.annual_rate * 100:.1f}% APY)"
        )
    
    config.last_calculated_at = datetime.utcnow()
    db.commit()
```

---

## Recurring Debits

### Debit Processor

```python
async def process_recurring_debits():
    """
    Process all due recurring debits.
    Runs daily at midnight.
    """
    today = date.today()
    
    due_debits = db.query(RecurringDebit).filter(
        RecurringDebit.is_active == True,
        RecurringDebit.next_debit_date <= today
    ).all()
    
    for debit in due_debits:
        try:
            await process_single_debit(debit)
        except InsufficientFundsError:
            # Notify parent of failed debit
            send_notification(
                user_id=get_parent_id(debit.family_id),
                type='DEBIT_FAILED',
                message=f"Recurring debit '{debit.name}' failed due to insufficient funds"
            )
        except Exception as e:
            logger.error(f"Failed to process debit {debit.id}: {e}")

async def process_single_debit(debit: RecurringDebit):
    """
    Process a single recurring debit.
    """
    account = get_account(debit.account_id)
    
    # Check balance (respect negative balance settings)
    family = get_family(debit.family_id)
    if not can_debit(account, debit.amount, family.settings):
        raise InsufficientFundsError()
    
    create_transaction(
        account_id=account.id,
        amount=-debit.amount,
        transaction_type='RECURRING_DEBIT',
        description=debit.name
    )
    
    debit.last_debit_date = date.today()
    debit.next_debit_date = calculate_next_debit_date(debit)
    
    db.commit()
```

---

## CSV Export

### Export Endpoint

```python
import csv
from io import StringIO
from fastapi.responses import StreamingResponse

@router.get("/transactions/export")
async def export_transactions(
    account_id: Optional[UUID] = None,
    start_date: Optional[date] = None,
    end_date: Optional[date] = None,
    context: FamilyContext = Depends(get_current_family_context),
    db: Session = Depends(get_db)
):
    """
    Export transactions to CSV.
    """
    query = db.query(Transaction).filter(Transaction.family_id == context.family_id)
    
    if account_id:
        query = query.filter(Transaction.account_id == account_id)
    if start_date:
        query = query.filter(Transaction.created_at >= start_date)
    if end_date:
        query = query.filter(Transaction.created_at <= end_date)
    
    transactions = query.order_by(Transaction.created_at.desc()).all()
    
    # Generate CSV
    output = StringIO()
    writer = csv.writer(output)
    
    # Header
    writer.writerow(['Date', 'Account', 'Type', 'Description', 'Amount', 'Balance After'])
    
    # Data
    for tx in transactions:
        writer.writerow([
            tx.created_at.strftime('%Y-%m-%d %H:%M'),
            tx.account.name,
            tx.transaction_type,
            tx.description,
            f"${tx.amount:.2f}",
            f"${tx.balance_after:.2f}"
        ])
    
    output.seek(0)
    
    filename = f"transactions_{context.family_id}_{date.today().isoformat()}.csv"
    
    return StreamingResponse(
        iter([output.getvalue()]),
        media_type="text/csv",
        headers={"Content-Disposition": f"attachment; filename={filename}"}
    )
```

---

## API Endpoints

### Allowances

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/allowances` | GET | List all allowances for family |
| `/api/v1/allowances` | POST | Create new allowance |
| `/api/v1/allowances/{id}` | GET | Get allowance details |
| `/api/v1/allowances/{id}` | PUT | Update allowance |
| `/api/v1/allowances/{id}` | DELETE | Deactivate allowance |
| `/api/v1/allowances/{id}/splits` | GET | Get allowance splits |
| `/api/v1/allowances/{id}/splits` | PUT | Update allowance splits |
| `/api/v1/allowances/{id}/process` | POST | Manually trigger payment |

### Interest

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/accounts/{id}/interest` | GET | Get interest config |
| `/api/v1/accounts/{id}/interest` | PUT | Set/update interest config |
| `/api/v1/accounts/{id}/interest/preview` | GET | Preview interest earnings |

### Recurring Debits

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/recurring-debits` | GET | List recurring debits |
| `/api/v1/recurring-debits` | POST | Create recurring debit |
| `/api/v1/recurring-debits/{id}` | PUT | Update recurring debit |
| `/api/v1/recurring-debits/{id}` | DELETE | Cancel recurring debit |

### Export

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/transactions/export` | GET | Export transactions to CSV |
| `/api/v1/reports/weekly-summary` | GET | Get weekly summary data |

---

## Frontend Components

### Split Builder UI

```jsx
function SplitBuilder({ accounts, value, onChange }) {
  const [splits, setSplits] = useState(value || []);
  
  const updateSplit = (accountId, percentage) => {
    const newSplits = splits.map(s => 
      s.accountId === accountId ? { ...s, percentage } : s
    );
    setSplits(newSplits);
    onChange(newSplits);
  };
  
  const total = splits.reduce((sum, s) => sum + s.percentage, 0);
  
  return (
    <div className="split-builder">
      {accounts.map(account => (
        <div key={account.id} className="split-row">
          <span>{account.name}</span>
          <input
            type="range"
            min="0"
            max="100"
            value={splits.find(s => s.accountId === account.id)?.percentage || 0}
            onChange={(e) => updateSplit(account.id, parseInt(e.target.value))}
          />
          <span>{splits.find(s => s.accountId === account.id)?.percentage || 0}%</span>
        </div>
      ))}
      <div className={`total ${total !== 100 ? 'error' : ''}`}>
        Total: {total}% {total !== 100 && '(must equal 100%)'}
      </div>
    </div>
  );
}
```

### Allowance Form

```jsx
function AllowanceForm({ kid, onSubmit }) {
  const { register, handleSubmit, watch } = useForm();
  const amountType = watch('amountType');
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Input label="Name" {...register('name')} placeholder="Weekly allowance" />
      
      <RadioGroup label="Amount Type" {...register('amountType')}>
        <Radio value="fixed">Fixed amount</Radio>
        <Radio value="formula">Age-based formula</Radio>
      </RadioGroup>
      
      {amountType === 'fixed' ? (
        <CurrencyInput label="Amount" {...register('amount')} />
      ) : (
        <Input 
          label="Formula" 
          {...register('formula')} 
          placeholder="age * 2"
          helperText="Use 'age' variable. Examples: 'age * 2', 'age * 1.5 + 5'"
        />
      )}
      
      <Select label="Frequency" {...register('frequency')}>
        <option value="weekly">Weekly</option>
        <option value="biweekly">Every 2 weeks</option>
        <option value="semimonthly">1st and 15th</option>
        <option value="monthly">Monthly</option>
      </Select>
      
      <SplitBuilder 
        accounts={kid.accounts}
        {...register('splits')}
      />
      
      <Button type="submit">Create Allowance</Button>
    </form>
  );
}
```

---

## Background Jobs Configuration

```python
# APScheduler job configuration for Phase 4

scheduler.add_job(
    func=process_allowance_payments,
    trigger="cron",
    hour=0,
    minute=0,
    id="allowance_processor",
    name="Process daily allowance payments"
)

scheduler.add_job(
    func=calculate_interest,
    trigger="cron",
    day=1,
    hour=0,
    minute=0,
    id="interest_calculator",
    name="Calculate monthly interest"
)

scheduler.add_job(
    func=process_recurring_debits,
    trigger="cron",
    hour=0,
    minute=5,
    id="recurring_debit_processor",
    name="Process recurring debits"
)

scheduler.add_job(
    func=generate_weekly_reports,
    trigger="cron",
    day_of_week="sun",
    hour=8,
    minute=0,
    id="weekly_report_generator",
    name="Generate weekly email reports"
)
```

---

## Testing Requirements

### Backend Tests

```python
def test_allowance_payment_fixed():
    """Test fixed amount allowance payment."""
    allowance = create_allowance(amount=10.00, amount_type='fixed')
    process_single_allowance(allowance)
    
    account = get_primary_account(allowance.user_id)
    assert account.balance == Decimal('10.00')

def test_allowance_payment_with_splits():
    """Test allowance with 60/20/20 split."""
    allowance = create_allowance(amount=10.00)
    create_splits(allowance.id, [
        ('spending', 60),
        ('saving', 20),
        ('giving', 20)
    ])
    
    process_single_allowance(allowance)
    
    assert get_account('spending').balance == Decimal('6.00')
    assert get_account('saving').balance == Decimal('2.00')
    assert get_account('giving').balance == Decimal('2.00')

def test_age_based_formula():
    """Test age-based allowance formula."""
    kid = create_kid(birthdate=date(2018, 5, 15))  # 8 years old
    allowance = create_allowance(formula='age * 2', amount_type='formula', user_id=kid.id)
    
    process_single_allowance(allowance)
    
    account = get_primary_account(kid.id)
    assert account.balance == Decimal('16.00')  # 8 * 2

def test_compound_interest():
    """Test monthly compound interest calculation."""
    account = create_account(balance=Decimal('100.00'))
    create_interest_config(account.id, annual_rate=0.12)  # 12% APY
    
    apply_interest(get_interest_config(account.id))
    
    # Monthly interest = 100 * (0.12/12) = $1.00
    assert account.balance == Decimal('101.00')

def test_interest_cap():
    """Test interest cap limits."""
    account = create_account(balance=Decimal('1000.00'))
    create_interest_config(account.id, annual_rate=0.12, cap_amount=Decimal('500.00'))
    
    apply_interest(get_interest_config(account.id))
    
    # Interest only on capped amount: 500 * (0.12/12) = $5.00
    assert account.balance == Decimal('1005.00')
```

### Coverage Requirements

- Allowance payments: 100%
- Interest calculations: 100%
- Split percentage validation: 100%
- CSV export: 80%
- Timezone handling: 100%

---

## Definition of Done

### Must Have
- [ ] Weekly, bi-weekly, monthly, semi-monthly allowances working
- [ ] Age-based formula evaluation
- [ ] Split deposits (60/20/20)
- [ ] Monthly interest calculation with caps
- [ ] Recurring debits
- [ ] CSV transaction export
- [ ] All tests passing with 100% coverage on financial operations

### Nice to Have
- [ ] Interest projector (show future balance)
- [ ] Allowance history view
- [ ] Skip next allowance option
- [ ] Debit pause/resume

---

## Dependencies

- **Requires**: Phase 1 (accounts, transactions)
- **Optional**: Phase 2-3 (can run in parallel)
- **Enables**: Phase 5 (AWS deployment includes job scheduling)

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Incorrect interest calculations | Extensive tests with known values |
| Timezone issues with payments | Store dates UTC, process in family timezone |
| Split rounding errors | Always round to 2 decimal places, adjust last split |
| Formula injection | Whitelist allowed operators and variables |

---

*Phase 4 Document - Last Updated: February 2026*
