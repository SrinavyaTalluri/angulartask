# angulartask

Using angular, AWS services like either Lambda or s3 bucket or appsync or dynamodb or combination of these Aws serivces and also springboot i want to built an Digital payment application the main requirements are 1)it should have 3 pages one is for login page ,second page is for add money ,third page is for history, fund transfer2)it should maintain local time3)The login page is like administer or user login page and in add money page per day only 10 thousand can be added into the account and user can withdraw when there is not less than 0 4) should allow to tranfer funds from a to b accounts

can you give the frontend code itself using angular ngrx and also make sure you explain me the flow

🏗️ ARCHITECTURE OVERVIEW
This is a Redux-based Angular application using nGRX for centralized state management. Think of it as a single store (database in memory) that manages all application data.
┌─────────────────────────────────────────────────────────────┐
│                    CENTRAL STATE STORE (nGRX)                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │    AUTH      │  │  TRANSACTION │  │   ACCOUNT    │       │
│  │  SLICE       │  │   SLICE      │  │   SLICE      │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
         ▲                    ▲                   ▲
         │ SELECT             │ SELECT             │ SELECT
         │ (Get Data)         │ (Get Data)         │ (Get Data)
         │                    │                    │
    ┌────┴────┐          ┌────┴────┐          ┌────┴────┐
    │ LOGIN   │          │DASHBOARD│          │TRANSFER │
    │COMPONENT│          │COMPONENT│          │COMPONENT│
    └────┬────┘          └────┬────┘          └────┬────┘
         │ DISPATCH            │ DISPATCH          │ DISPATCH
         │ (Send Action)       │ (Send Action)     │ (Send Action)
         │                     │                   │
    ┌────┴─────────┬───────────┴───────────┬───────┴────┐
    │              │                       │            │
 ACTIONS       EFFECTS                  REDUCERS     SELECTORS
 (Events)      (Side Effects)           (Update     (Extract
               (API Calls,              State)      Data)
               Local Storage)
📦 THREE MAIN STORE SLICES
1. AUTH STORE (User Login/Logout)
What it manages: User authentication state
AUTH STATE:
{
  user: { id, username, email, role, accountNumber, createdAt },
  isAuthenticated: true/false,
  loading: true/false,
  error: null or "error message",
  lastLoginTime: Date
}

Flow when user logs in:
1. User types "admin" in username and "admin123" in password
   ↓
2. Clicks "LOGIN" button
   ↓
3. Component dispatches: AuthActions.login({ username, password })
   ↓
4. AuthEffects listens for the 'login' action
   ↓
5. AuthEffects calls: authService.login() → Validates with mockUsers
   ↓
6. If credentials match → AuthEffects dispatches: loginSuccess({ user })
   ↓
7. authReducer updates state:
   - Sets user data
   - Sets isAuthenticated = true
   - Sets loading = false
   - Saves to localStorage
   ↓
8. Component's Observable (user$, isAuthenticated$) receives new data
   ↓
9. Component automatically updates UI
   ↓
10. Component routes to /dashboard


code flow
// STEP 1: User fills form and clicks submit
onSubmit(): void {
  if (this.loginForm.valid) {
    this.store.dispatch(
      AuthActions.login({
        payload: this.loginForm.value  // { username: 'admin', password: 'admin123' }
      })
    );
  }
}

// STEP 2: Effects intercepts the action
login$ = createEffect(() =>
  this.actions$.pipe(
    ofType(AuthActions.login),  // Listen for 'login' action
    mergeMap(action =>
      this.authService.login(action.payload).pipe(  // Call service
        map(user => AuthActions.loginSuccess({ user })),  // If success
        catchError(error => of(AuthActions.loginFailure({ error: error.message })))  // If error
      )
    )
  )
);

// STEP 3: Reducer updates the state
on(AuthActions.loginSuccess, (state, { user }) => ({
  ...state,  // Keep existing state
  user: user,  // Update user
  isAuthenticated: true,  // Mark as authenticated
  loading: false,
  error: null,
  lastLoginTime: new Date()
}))

// STEP 4: Component selects from store
isAuthenticated$ = this.store.select(selectIsAuthenticated);
// This subscribes to state changes automatically

Demo credentials
admin / admin123    → Full access (Admin role)
user1 / user123     → Limited access (User role)
user2 / user123     → Limited access (User role)

2. TRANSACTION STORE (Add Money, Withdraw, Transfer)
What it manages: All financial transactions and daily limits
TRANSACTION STATE:
{
  transactions: [
    { 
      id, 
      fromAccountId, 
      toAccountId, 
      amount, 
      type: 'DEPOSIT' | 'WITHDRAWAL' | 'TRANSFER',
      description,
      timestamp,
      status: 'SUCCESS' | 'FAILED'
    }
  ],
  dailyLimitRemaining: 8500,  // Out of ₹10,000
  loading: true/false,
  error: null
}

⚠️ BUSINESS LOGIC - Daily Limit:

Each user has a ₹10,000 daily limit
This resets at midnight (00:00)
Applies to: Deposits + Withdrawals + Transfers combined
Flow Example:

SCENARIO: User adds ₹5,000 to account
1. User enters 5000 in "Add Money" form
   ↓
2. Clicks "ADD" button
   ↓
3. Component dispatches: TransactionActions.addMoney({
     accountId: 'ACC_USER_001',
     amount: 5000,
     description: 'Added via app'
   })
   ↓
4. TransactionEffects listens for 'addMoney' action
   ↓
5. Calls: transactionService.addMoney(5000) which:
   - Checks if amount < remaining daily limit (10000 - spent)
   - If valid → Creates transaction object
   - Updates account balance
   - Returns transaction
   ↓
6. If success → Dispatches: addMoneySuccess({ transaction })
   If error (exceeds limit) → Dispatches: addMoneyFailure({ error })
   ↓
7. Reducer updates transaction state:
   - Adds transaction to array
   - Updates dailyLimitRemaining
   ↓
8. Component's Observables update:
   - transactions$ shows new transaction
   - dailyLimitRemaining$ updates progress bar
   - currentBalance$ updates balance

   Code Example

   // Dashboard Component
onAddMoney(): void {
  if (this.addMoneyForm.valid && this.accountNumber$) {
    this.accountNumber$.subscribe(accountNumber => {
      this.store.dispatch(
        TransactionActions.addMoney({
          accountId: accountNumber,
          amount: this.addMoneyForm.value.amount,
          description: this.addMoneyForm.value.description
        })
      );
    });
  }
}

// Effects intercepts
addMoney$ = createEffect(() =>
  this.actions$.pipe(
    ofType(TransactionActions.addMoney),
    mergeMap(action =>
      this.transactionService.addMoney(
        action.accountId,
        action.amount,
        action.description
      ).pipe(
        map(transaction => 
          TransactionActions.addMoneySuccess({ transaction })
        ),
        catchError(error => 
          of(TransactionActions.addMoneyFailure({ 
            error: error.message 
          }))
        )
      )
    )
  )
);

// Service validates daily limit
addMoney(accountId: string, amount: number, description: string): Observable<Transaction> {
  const remainingLimit = this.getRemainingDailyLimit(accountId);
  
  if (amount > remainingLimit) {
    return throwError(new Error(
      `Exceeds daily limit. Remaining: ₹${remainingLimit}`
    ));
  }
  
  const transaction = {
    id: generateId(),
    fromAccountId: accountId,
    toAccountId: accountId,
    amount: amount,
    type: 'DEPOSIT',
    description: description,
    timestamp: new Date(),
    status: 'SUCCESS'
  };
  
  this.transactions.push(transaction);
  return of(transaction);
}

3. ACCOUNT STORE (Balance Management)
What it manages: Account balances and account information
ACCOUNT STATE:
{
  currentAccount: {
    accountNumber: 'ACC_USER_001',
    balance: 50000,
    owner: 'user1',
    createdAt: Date
  },
  accounts: [ /* all accounts */ ],
  loading: true/false,
  error: null
}

Flow when dashboard loads:
1. Dashboard component initializes
   ↓
2. Dispatches: loadAccountByNumber({ accountNumber: 'ACC_USER_001' })
   ↓
3. Effects calls: accountService.getAccountByNumber()
   ↓
4. Service returns account data from mock storage
   ↓
5. Effects dispatches: loadAccountByNumberSuccess({ account })
   ↓
6. Reducer updates currentAccount
   ↓
7. Selector selectCurrentAccountBalance returns balance
   ↓
8. Component's currentBalance$ Observable receives updated balance
   ↓
9. Template displays: "Your Balance: ₹50,000"

🔄 COMPLETE USER JOURNEY WITH ALL STORES
Scenario: User transfers ₹2,000 from user1 to user2
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: USER LOGS IN (AUTH STORE)                               │
├─────────────────────────────────────────────────────────────────┤
│ User enters: username='user1', password='user123'                │
│ ↓                                                                │
│ Action: AuthActions.login()                                     │
│ ↓                                                                │
│ AuthEffects validates → Success                                 │
│ ↓                                                                │
│ AuthReducer: user = { id, username, accountNumber='ACC_USER_001'} │
│ Result: user$ Observable updates → Routes to /dashboard         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ STEP 2: DASHBOARD LOADS (AUTH + ACCOUNT + TRANSACTION STORES)   │
├─────────────────────────────────────────────────────────────────┤
│ Actions dispatched:                                              │
│  a) loadAccountByNumber({ accountNumber: 'ACC_USER_001' })      │
│  b) loadTransactionHistory({ accountId: 'ACC_USER_001' })       │
│  c) loadDailyLimit({ accountId: 'ACC_USER_001' })               │
│ ↓                                                                │
│ ACCOUNT STORE:                                                   │
│  - Fetches: { accountNumber, balance: 50000 }                   │
│  - Selectors return balance to component                        │
│  - Component displays: "Balance: ₹50,000"                       │
│ ↓                                                                │
│ TRANSACTION STORE:                                               │
│  - Loads previous transactions                                   │
│  - Calculates: dailyLimit = 10000 - (spent today)               │
│  - Selectors return dailyLimitRemaining: 10000                  │
│  - Component displays progress bar: 0/10000 spent               │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ STEP 3: USER CLICKS "TRANSFER" (TRANSACTION STORE)              │
├─────────────────────────────────────────────────────────────────┤
│ User fills form:                                                 │
│  - Recipient: user2 (ACC_USER_002)                              │
│  - Amount: 2000                                                 │
│ ↓                                                                │
│ Action: TransactionActions.transferFunds({                      │
│   fromAccountId: 'ACC_USER_001',                                │
│   toAccountId: 'ACC_USER_002',                                  │
│   amount: 2000,                                                 │
│   description: 'Transfer to user2',                             │
│   currentBalance: 50000                                         │
│ })                                                               │
│ ↓                                                                │
│ TransactionEffects calls service:                               │
│  transactionService.transferFunds()                             │
│ ↓                                                                │
│ Service validates:                                              │
│  ✓ Amount (2000) < dailyLimit (10000)? YES                     │
│  ✓ Balance (50000) >= Amount (2000)? YES                       │
│  ✓ Valid → Creates transaction                                 │
│ ↓                                                                │
│ Action: TransactionActions.transferFundsSuccess({               │
│   transaction: { id, fromAccountId, toAccountId, amount, ... } │
│ })                                                               │
│ ↓                                                                │
│ TRANSACTION STORE updates:                                       │
│  - transactions[] += new transaction                             │
│  - dailyLimitRemaining = 10000 - 2000 = 8000                   │
│ ↓                                                                │
│ ACCOUNT STORE updates (ACCOUNT EFFECTS):                        │
│  - ACC_USER_001 balance: 50000 → 48000                         │
│  - ACC_USER_002 balance: 75000 → 77000                         │
│ ↓                                                                │
│ Components update via Selectors:                                 │
│  - currentBalance$: 48000                                       │
│  - dailyLimitRemaining$: 8000                                   │
│  - Progress bar: 2000/10000 spent                               │
│ ↓                                                                │
│ User sees:                                                       │
│  "✓ Transfer successful! Balance: ₹48,000"                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ STEP 4: USER CHECKS HISTORY (TRANSACTION STORE)                 │
├─────────────────────────────────────────────────────────────────┤
│ Clicks "History" tab                                             │
│ ↓                                                                │
│ Component selects: selectAllTransactions                        │
│ ↓                                                                │
│ Selector returns all transactions for this user:                │
│  [                                                               │
│   { type: 'DEPOSIT', amount: 5000, description: '...' },       │
│   { type: 'TRANSFER', amount: 2000, toAccountId: '...' },      │
│   ...                                                           │
│  ]                                                               │
│ ↓                                                                │
│ Component displays filtered, sorted transaction list            │
└─────────────────────────────────────────────────────────────────┘
🎯 KEY CONCEPTS EXPLAINED

1. ACTIONS (Events)
// Action = "Something happened" event
AuthActions.login({ payload: { username, password } })
// Meaning: "A login attempt was made"

TransactionActions.addMoney({ accountId, amount, description })
// Meaning: "A user tried to add money"

TransactionActions.addMoneySuccess({ transaction })
// Meaning: "Adding money succeeded"

TransactionActions.addMoneyFailure({ error })
// Meaning: "Adding money failed"

2. REDUCERS (Pure Functions)
// Reducer = "Update the state based on action"
on(AuthActions.loginSuccess, (state, { user }) => ({
  ...state,  // Keep everything else
  user: user,  // Change only this
  isAuthenticated: true,  // Change only this
  loading: false  // Change only this
}))

// In Excel terms:
// IF action is "loginSuccess" THEN
//   SET state.user = action.user
//   SET state.isAuthenticated = true
//   SET state.loading = false

3. EFFECTS (Side Effects)
// Effect = "Listen for action and do something with external world"
login$ = createEffect(() =>
  this.actions$.pipe(
    ofType(AuthActions.login),  // Listen for this action
    mergeMap(action =>
      this.authService.login(action.payload).pipe(  // Call external API/service
        map(user => AuthActions.loginSuccess({ user })),  // Dispatch success action
        catchError(error => of(AuthActions.loginFailure({ error })))  // Or failure action
      )
    )
  )
);

// In English:
// WHEN I hear "login" action:
//   CALL authService.login()
//   IF it succeeds: DISPATCH loginSuccess
//   IF it fails: DISPATCH loginFailure

4. SELECTORS (Extract Data)
// Selector = "Get a specific piece of state"
export const selectUser = (state: AppState) => state.auth.user;
export const selectIsAuthenticated = (state: AppState) => state.auth.isAuthenticated;
export const selectCurrentBalance = (state: AppState) => state.account.currentAccount.balance;

// In component:
user$ = this.store.select(selectUser);
// This is an Observable that updates whenever state.auth.user changes
// Template: {{ user$ | async | json }}

📱 COMPONENTS BREAKDOWN

LoginComponent
USER INTERACTION:
1. Enters username & password
2. Clicks "LOGIN" or fills demo credentials
   ↓
COMPONENT LOGIC:
3. Validates form (minLength, pattern)
4. Dispatches AuthActions.login()
5. Subscribes to loading$ and error$ observables
6. Shows loading spinner while loading$ = true
7. Shows error message if error$ has value
8. Routes to dashboard if isAuthenticated$ = true
   ↓
TEMPLATE ELEMENTS:
- Input fields with validation errors
- Demo buttons (auto-fill credentials)
- Password visibility toggle
- Loading spinner
- Error message display

DashboardComponent
SHOWS:
- Current balance (from ACCOUNT store)
- Daily limit remaining (from TRANSACTION store)
- Current time (updates every second)
- Two tabs: "Add Money" / "Withdraw Money"

TAB 1: ADD MONEY
- Form: Enter amount (₹100-₹10,000)
- Action: dispatchTransactionActions.addMoney()
- Validation: Checks daily limit in service
- Success: Shows confirmation, updates balance

TAB 2: WITHDRAW MONEY
- Form: Enter amount
- Action: dispatch TransactionActions.withdrawMoney()
- Validation: Must have sufficient balance
- Success: Deducts from balance

TransferComponent
SHOWS:

- Recipient dropdown (list of other users)
- Amount input
- Current balance
- Daily limit remaining

ON SUBMIT:
1. Validates recipient is different from sender
2. Validates amount < balance
3. Validates amount < remaining daily limit
4. Dispatches transferFunds()
   ↓
FLOW:
- Effect calls transactionService.transferFunds()
- Service updates BOTH accounts:
  a) Deducts from sender
  b) Adds to recipient
- Success → Show confirmation
- Link to "History" to see transaction

HistoryComponent
SHOWS:
- List of all transactions
- Filter buttons: All / Deposits / Withdrawals / Transfers
- Details: Amount, Date, Description, Type

DATA FLOW:
1. Component loads
2. Selects selectAllTransactions from store
3. Filters based on selected filter
4. Sorts by timestamp (newest first)
5. Displays in table

INTERACTION:
- Click filter button → Array.filter() → Display filtered list
- No API call needed (already in store)

🔐 ROUTE PROTECTION (AuthGuard)
// AuthGuard ensures only logged-in users can access protected routes
export class AuthGuard implements CanActivate {
  canActivate(route: ActivatedRouteSnapshot): Observable<boolean> {
    // Check if user is authenticated
    return this.store.select(selectIsAuthenticated).pipe(
      map(isAuth => {
        if (isAuth) {
          return true;  // Allow access
        } else {
          this.router.navigate(['/login']);  // Redirect to login
          return false;  // Block access
        }
      })
    );
  }
}

// In routing:
{
  path: 'dashboard',
  component: DashboardComponent,
  canActivate: [AuthGuard]  // Must be logged in
}

Routes:
/login - Public (no guard)
/dashboard - Protected (requires auth)
/transfer - Protected (requires auth)
/history - Protected (requires auth)

💾 LOCAL STORAGE PERSISTENCE
The app saves to browser localStorage:
localStorage.setItem('currentUser', JSON.stringify(user));
localStorage.setItem('lastLoginTime', new Date().toISOString());

// On app reload:
// - Checks localStorage for currentUser
// - If found, auto-logs in user
// - If not found, redirects to login

⚙️ VALIDATION RULES
ADD MONEY:
  ✓ Amount: 100 - 10,000
  ✓ Daily limit not exceeded
  
WITHDRAW:
  ✓ Amount: 100 - balance
  ✓ Sufficient balance
  ✓ Daily limit not exceeded
  
TRANSFER:
  ✓ Recipient ≠ Sender
  ✓ Amount ≤ balance
  ✓ Daily limit not exceeded
  ✓ Valid recipient selection

  🚀 DATA FLOW SUMMARY
  USER ACTION
    ↓
COMPONENT
    ↓
DISPATCH ACTION
    ↓
EFFECTS (Optional: Call Service)
    ↓
DISPATCH SUCCESS/FAILURE ACTION
    ↓
REDUCER (Update State)
    ↓
SELECTOR (Extract Data from State)
    ↓
COMPONENT RECEIVES NEW DATA via Observable
    ↓
TEMPLATE UPDATES (via async pipe)
    ↓
USER SEES RESULT
