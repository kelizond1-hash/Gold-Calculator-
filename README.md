<!DOCTYPE html>

<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="apple-mobile-web-app-title" content="Gold Calculator">
    <title>Gold Business Calculator</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.5/babel.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        * { -webkit-tap-highlight-color: transparent; }
        body { overscroll-behavior: none; }
    </style>
</head>
<body>
    <div id="root"></div>
    <script type="text/babel">
        const { useState, useMemo } = React;

```
    const GoldCalculator = () => {
      const [buyPrice, setBuyPrice] = useState(85000);
      const [sellPrice, setSellPrice] = useState(135000);
      const [opCost10kg, setOpCost10kg] = useState(15000);
      const [opCost25kg, setOpCost25kg] = useState(18000);
      const [opCost50kg, setOpCost50kg] = useState(20000);
      const [opCost100kg, setOpCost100kg] = useState(25000);
      const [monthlyExpenses, setMonthlyExpenses] = useState(25000);
      const [scenario1Pct, setScenario1Pct] = useState(10);
      const [scenario2Pct, setScenario2Pct] = useState(20);
      const [activeScenario, setActiveScenario] = useState('both');
      const [showInputs, setShowInputs] = useState(false);

      const trancheSchedule = [
        { month: 1, kgs: 10 }, { month: 1, kgs: 10 },
        { month: 2, kgs: 10 }, { month: 2, kgs: 10 },
        { month: 3, kgs: 10 }, { month: 3, kgs: 25 },
        { month: 4, kgs: 25 }, { month: 4, kgs: 25 },
        { month: 5, kgs: 25 }, { month: 5, kgs: 50 },
        { month: 6, kgs: 50 }, { month: 6, kgs: 50 },
        { month: 7, kgs: 50 }, { month: 7, kgs: 100 },
        { month: 8, kgs: 100 }, { month: 8, kgs: 100 },
        { month: 9, kgs: 100 }, { month: 9, kgs: 100 },
        { month: 10, kgs: 100 }, { month: 10, kgs: 100 },
        { month: 11, kgs: 100 }, { month: 11, kgs: 100 },
        { month: 12, kgs: 100 }, { month: 12, kgs: 100 },
      ];

      const getOpCost = (kgs) => {
        if (kgs === 10) return opCost10kg;
        if (kgs === 25) return opCost25kg;
        if (kgs === 50) return opCost50kg;
        if (kgs === 100) return opCost100kg;
        return 0;
      };

      const calculateTransactions = (investorPct, payUntil5x = false) => {
        const INITIAL_CAPITAL = 1000000;
        const TARGET_5X = 6000000;
        let cash = INITIAL_CAPITAL;
        let totalPaidToInvestor = 0;
        let transactions = [];
        let quarterlyOwedToInvestor = 0;
        let monthlyExpensesPaid = {};
        
        for (let i = 0; i < trancheSchedule.length; i++) {
          const { month, kgs } = trancheSchedule[i];
          const opCost = getOpCost(kgs);
          const buyCost = kgs * buyPrice;
          const sellRevenue = kgs * sellPrice;
          const grossProfit = sellRevenue - buyCost;
          const netProfit = grossProfit - opCost;

          if (cash < buyCost + opCost) break;

          const cashBeforeBuy = cash;
          cash -= buyCost + opCost;
          cash += sellRevenue;

          let expensesPulled = 0;
          if (!monthlyExpensesPaid[month]) {
            cash -= monthlyExpenses;
            expensesPulled = monthlyExpenses;
            monthlyExpensesPaid[month] = true;
          }

          const investorShareThisTranche = netProfit * (investorPct / 100);
          quarterlyOwedToInvestor += investorShareThisTranche;

          let investorPayment = 0;
          let isQuarterEnd = month % 3 === 0 && (i === trancheSchedule.length - 1 || trancheSchedule[i + 1].month !== month);

          if (isQuarterEnd) {
            if (payUntil5x && totalPaidToInvestor >= TARGET_5X) {
              investorPayment = 0;
              quarterlyOwedToInvestor = 0;
            } else {
              investorPayment = quarterlyOwedToInvestor;
              if (payUntil5x && totalPaidToInvestor + investorPayment > TARGET_5X) {
                investorPayment = TARGET_5X - totalPaidToInvestor;
              }
              cash -= investorPayment;
              totalPaidToInvestor += investorPayment;
              quarterlyOwedToInvestor = 0;
            }
          }

          transactions.push({
            id: i + 1, month, kgs, buyCost, sellRevenue, grossProfit, opCost,
            netProfit, investorShareThisTranche, expensesPulled,
            investorPayment, totalPaidToInvestor, cashFinal: cash, isQuarterEnd
          });
        }
        return { transactions, totalPaidToInvestor, finalCash: cash };
      };

      const scenario1 = useMemo(() => calculateTransactions(scenario1Pct, false), [
        buyPrice, sellPrice, opCost10kg, opCost25kg, opCost50kg, opCost100kg, monthlyExpenses, scenario1Pct
      ]);

      const scenario2 = useMemo(() => calculateTransactions(scenario2Pct, true), [
        buyPrice, sellPrice, opCost10kg, opCost25kg, opCost50kg, opCost100kg, monthlyExpenses, scenario2Pct
      ]);

      const formatCurrency = (num) => {
        return new Intl.NumberFormat('en-US', {
          style: 'currency', currency: 'USD',
          minimumFractionDigits: 0, maximumFractionDigits: 0
        }).format(num);
      };

      const ScenarioSummary = ({ data, title, color }) => {
        const totalGrossProfit = data.transactions.reduce((sum, t) => sum + t.grossProfit, 0);
        const totalNetProfit = data.transactions.reduce((sum, t) => sum + t.netProfit, 0);
        const totalOpCosts = data.transactions.reduce((sum, t) => sum + t.opCost, 0);
        const totalExpenses = data.transactions.reduce((sum, t) => sum + t.expensesPulled, 0);
        const yourNet = totalNetProfit - data.totalPaidToInvestor - totalExpenses;

        return (
          <div className={`bg-white rounded-lg shadow-lg p-4 border-t-4 ${color}`}>
            <h3 className="text-lg font-bold mb-3">{title}</h3>
            <div className="space-y-2 text-sm">
              <div className="flex justify-between">
                <span className="text-gray-600">Gross Profit:</span>
                <span className="font-semibold text-green-600">{formatCurrency(totalGrossProfit)}</span>
              </div>
              <div className="flex justify-between">
                <span className="text-gray-600">Op Costs:</span>
                <span className="font-semibold text-red-600">-{formatCurrency(totalOpCosts)}</span>
              </div>
              <div className="flex justify-between">
                <span className="text-gray-600">Expenses:</span>
                <span className="font-semibold text-red-600">-{formatCurrency(totalExpenses)}</span>
              </div>
              <div className="flex justify-between border-t pt-2">
                <span className="text-gray-600">Net Profit:</span>
                <span className="font-bold text-green-700">{formatCurrency(totalNetProfit)}</span>
              </div>
              <div className="flex justify-between">
                <span className="text-gray-600">To Investor:</span>
                <span className="font-semibold text-blue-600">{formatCurrency(data.totalPaidToInvestor)}</span>
              </div>
              <div className="flex justify-between border-t pt-2">
                <span className="text-gray-600">Your Net:</span>
                <span className="font-bold text-green-700">{formatCurrency(yourNet)}</span>
              </div>
              <div className="flex justify-between border-t pt-2">
                <span className="text-gray-600">Final Cash:</span>
                <span className="text-xl font-bold text-blue-700">{formatCurrency(data.finalCash)}</span>
              </div>
            </div>
          </div>
        );
      };

      return (
        <div className="max-w-7xl mx-auto p-3 bg-gray-50 min-h-screen">
          <div className="mb-4">
            <h1 className="text-2xl font-bold text-gray-800">Gold Business Calculator</h1>
            <p className="text-sm text-gray-600">12-Month Forecast</p>
          </div>

          <button
            onClick={() => setShowInputs(!showInputs)}
            className="w-full mb-4 px-4 py-3 bg-blue-600 text-white rounded-lg font-medium"
          >
            {showInputs ? '✕ Close Settings' : '⚙️ Adjust Parameters'}
          </button>

          {showInputs && (
            <div className="bg-white rounded-lg shadow-lg p-4 mb-4">
              <div className="grid grid-cols-2 gap-3 text-sm">
                <div>
                  <label className="block font-medium text-gray-700 mb-1">Buy Price/kg</label>
                  <input type="number" value={buyPrice} onChange={(e) => setBuyPrice(Number(e.target.value))}
                    className="w-full px-2 py-2 border rounded" />
                </div>
                <div>
                  <label className="block font-medium text-gray-700 mb-1">Sell Price/kg</label>
                  <input type="number" value={sellPrice} onChange={(e) => setSellPrice(Number(e.target.value))}
                    className="w-full px-2 py-2 border rounded" />
                </div>
                <div>
                  <label className="block font-medium text-gray-700 mb-1">Monthly Expenses</label>
                  <input type="number" value={monthlyExpenses} onChange={(e) => setMonthlyExpenses(Number(e.target.value))}
                    className="w-full px-2 py-2 border rounded" />
                </div>
                <div>
                  <label className="block font-medium text-gray-700 mb-1">Op Cost (10kg)</label>
                  <input type="number" value={opCost10kg} onChange={(e) => setOpCost10kg(Number(e.target.value))}
                    className="w-full px-2 py-2 border rounded" />
                </div>
                <div>
                  <label className="block font-medium text-gray-700 mb-1">Op Cost (25kg)</label>
                  <input type="number" value={opCost25kg} onChange={(e) => setOpCost25kg(Number(e.target.value))}
                    className="w-full px-2 py-2 border rounded" />
                </div>
                <div>
                  <label className="block font-medium text-gray-700 mb-1">Op Cost (50kg)</label>
                  <input type="number" value={opCost50kg} onChange={(e) => setOpCost50kg(Number(e.target.value))}
                    className="w-full px-2 py-2 border rounded" />
                </div>
                <div>
                  <label className="block font-medium text-gray-700 mb-1">Op Cost (100kg)</label>
                  <input type="number" value={opCost100kg} onChange={(e) => setOpCost100kg(Number(e.target.value))}
                    className="w-full px-2 py-2 border rounded" />
                </div>
                <div>
                  <label className="block font-medium text-gray-700 mb-1">Scenario 1 %</label>
                  <input type="number" value={scenario1Pct} onChange={(e) => setScenario1Pct(Number(e.target.value))}
                    className="w-full px-2 py-2 border rounded" />
                </div>
                <div className="col-span-2">
                  <label className="block font-medium text-gray-700 mb-1">Scenario 2 % (Until 5X)</label>
                  <input type="number" value={scenario2Pct} onChange={(e) => setScenario2Pct(Number(e.target.value))}
                    className="w-full px-2 py-2 border rounded" />
                </div>
              </div>
            </div>
          )}

          <div className="flex gap-2 mb-4 overflow-x-auto">
            <button onClick={() => setActiveScenario('both')}
              className={`px-4 py-2 rounded-lg font-medium whitespace-nowrap ${activeScenario === 'both' ? 'bg-blue-600 text-white' : 'bg-gray-200'}`}>
              Compare
            </button>
            <button onClick={() => setActiveScenario('scenario1')}
              className={`px-4 py-2 rounded-lg font-medium whitespace-nowrap ${activeScenario === 'scenario1' ? 'bg-purple-600 text-white' : 'bg-gray-200'}`}>
              Scenario 1
            </button>
            <button onClick={() => setActiveScenario('scenario2')}
              className={`px-4 py-2 rounded-lg font-medium whitespace-nowrap ${activeScenario === 'scenario2' ? 'bg-green-600 text-white' : 'bg-gray-200'}`}>
              Scenario 2
            </button>
          </div>

          <div className="space-y-4">
            {(activeScenario === 'both' || activeScenario === 'scenario1') && (
              <ScenarioSummary data={scenario1}
                title={`Scenario 1: ${scenario1Pct}% Per Transaction`}
                color="border-purple-500" />
            )}
            {(activeScenario === 'both' || activeScenario === 'scenario2') && (
              <ScenarioSummary data={scenario2}
                title={`Scenario 2: ${scenario2Pct}% Until $6M`}
                color="border-green-500" />
            )}
          </div>
        </div>
      );
    };

    ReactDOM.render(<GoldCalculator />, document.getElementById('root'));
</script>
```

</body>
</html>