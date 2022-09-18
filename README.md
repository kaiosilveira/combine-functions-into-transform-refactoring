[![Continuous Integration](https://github.com/kaiosilveira/combine-functions-into-transform/actions/workflows/ci.yml/badge.svg)](https://github.com/kaiosilveira/combine-functions-into-transform/actions/workflows/ci.yml)

ℹ️ _This repository is part of my Refactoring catalog based on Fowler's book with the same title. Please see [kaiosilveira/refactoring](https://github.com/kaiosilveira/refactoring) for more details._

# Combine Functions into Transform

Sometimes we see groups of functions repeatedly operating together over a chunk of data. These functions may be independent and well defined, but their responsibilities are tightly related to some other, bigger aspect of a piece of computation. In this cases, we can create a higher order transform function to wrap this bigger aspect and keep all clients consistent.

## Working example

The working example for this refactoring is a small system that calculates the taxable charge for... tea (?). It contains helper functions to calculate the `baseRate` and the `taxThreshold`, but there is a lot of duplication when it comes to calculating the base charge inside the clients of this code. Our goal is combine these different aspects into a transform function that takes care of all the details and return an enhanced object.

### Before

The starting code for this example contains a `tax-utils` module with the utility functions mentioned above (`baseRate` and `taxThreshold`), it also contains the code for the clients themselves, which are shown below.

The `tax-utils` module:

```javascript
function baseRate(month, year) {
  return RATES_BY_MONTH_AND_YEAR[`${month}/${year}`];
}

function taxThreshold(year) {
  return THRESHOLDS_BY_YEAR[year];
}
```

Clients one and two have the logic for calculating the base charge duplicated:

```javascript
// client 1:
const aReading = acquireReading();
const baseCharge = baseRate(aReading.month, aReading.year) * aReading.quantity;

// client 2:
const aReading = acquireReading();
const base = baseRate(aReading.month, aReading.year) * aReading.quantity;
```

While client three has a method already created to do this job:

```javascript
// client 3
const aReading = acquireReading();
const basicChargeAmount = calculateBaseCharge(aReading);

console.log(`basic charge amount is ${basicChargeAmount}`);

function calculateBaseCharge(aReading) {
  return baseRate(aReading.month, aReading.year) * aReading.quantity;
}
```

### After

After finishing this refactoring the code looks quite different, but the core part is the new `enrichReading` function:

```javascript
function enrichReading(aReading) {
  const result = cloneDeep(aReading);
  result.baseCharge = calculateBaseCharge(result);
  result.taxableCharge = Math.max(0, result.baseCharge - taxThreshold(aReading.year));
  return result;
}
```

As we aimed for, we've implemented a function that creates a deep clone of the `reading` object and adds the `baseCharge` and the `taxableCharge` to it.

### Test suite

Writing a test suite for this example was hard, mainly because this domain doesn't quite make sense but also because the `baseRate` and `taxThreshold` functions were not well explored in the book (of course because they were not the focus). So a dummy logic was put in place, using lookup objects with fixed values.

For the `enrichReading` function, though, we have enough tests to make sure it behaves the way we expect. The test suite looks like this:

```javascript
describe('enrichReading', () => {
  const inputReading = { customer: 'Ivan', quantity: 10, month: 5, year: 2017 };

  it('should return a copy of the original object', () => {
    const resultingReading = enrichReading(inputReading);
    expect(resultingReading.customer).toEqual(inputReading.customer);
    expect(resultingReading.month).toEqual(inputReading.month);
    expect(resultingReading.year).toEqual(inputReading.year);
    expect(resultingReading.quantity).toEqual(inputReading.quantity);
  });

  it('should enrich the reading with the base charge', () => {
    const resultingReading = enrichReading(inputReading);
    expect(typeof resultingReading.baseCharge).toBe('number');
  });

  it('should enrich the reading with the taxable charge', () => {
    const resultingReading = enrichReading(inputReading);
    expect(typeof resultingReading.taxableCharge).toBe('number');
  });
});
```

### Steps

To perform this refactoring, we first introduce the `enrichReading` function, which will be responsible for adding the new fields we want to the `reading` object. To start with, this functions does nothing but creating and returning a clone of the original `reading` input object:

```diff
diff --git a/src/reading/index.js b/src/reading/index.js
@@ -1,7 +1,14 @@
+const { cloneDeep } = require('lodash');
+
 const reading = { customer: 'Ivan', quantity: 10, month: 5, year: 2017 };

 function acquireReading() {
   return reading;
 }

-module.exports = { acquireReading };
+function enrichReading(aReading) {
+  const result = cloneDeep(aReading);
+  return result;
+}
+
+module.exports = { acquireReading, enrichReading };

diff --git a/src/reading/index.test.js b/src/reading/index.test.js
@@ -0,0 +1,14 @@
+const { enrichReading } = require('.');
+
+describe('enrichReading', () => {
+  it('should return a copy of the original object', () => {
+    const inputReading = { customer: 'Ivan', quantity: 10, month: 5, year: 2017 };
+
+    const resultingReading = enrichReading(inputReading);
+
+    expect(resultingReading.customer).toEqual(inputReading.customer);
+    expect(resultingReading.month).toEqual(inputReading.month);
+    expect(resultingReading.year).toEqual(inputReading.year);
+    expect(resultingReading.quantity).toEqual(inputReading.quantity);
+  });
+});
```

Then, we update `client3.js` to use it:

```diff
diff --git a/src/client3.js b/src/client3.js
@@ -1,7 +1,8 @@
-const { acquireReading } = require('./reading');
+const { acquireReading, enrichReading } = require('./reading');
 const { baseRate } = require('./tax-utils');

-const aReading = acquireReading();
+const rawReading = acquireReading();
+const aReading = enrichReading(rawReading);
 const basicChargeAmount = calculateBaseCharge(aReading);

 console.log(`basic charge amount is ${basicChargeAmount}`);
```

With this in place, we can now enrich the `reading` with the `baseCharge` calculation in the `enrichReading` function:

```diff
diff --git a/src/reading/index.js b/src/reading/index.js
@@ -1,4 +1,5 @@
 const { cloneDeep } = require('lodash');
+const { baseRate } = require('../tax-utils');

 const reading = { customer: 'Ivan', quantity: 10, month: 5, year: 2017 };

@@ -8,7 +9,12 @@ function acquireReading() {

 function enrichReading(aReading) {
   const result = cloneDeep(aReading);
+  result.baseCharge = calculateBaseCharge(result);
   return result;
 }

+function calculateBaseCharge(aReading) {
+  return baseRate(aReading.month, aReading.year) * aReading.quantity;
+}
+
 module.exports = { acquireReading, enrichReading };

diff --git a/src/reading/index.test.js b/src/reading/index.test.js
@@ -1,14 +1,18 @@
 const { enrichReading } = require('.');

 describe('enrichReading', () => {
-  it('should return a copy of the original object', () => {
-    const inputReading = { customer: 'Ivan', quantity: 10, month: 5, year: 2017 };
+  const inputReading = { customer: 'Ivan', quantity: 10, month: 5, year: 2017 };

+  it('should return a copy of the original object', () => {
     const resultingReading = enrichReading(inputReading);
-
     expect(resultingReading.customer).toEqual(inputReading.customer);
     expect(resultingReading.month).toEqual(inputReading.month);
     expect(resultingReading.year).toEqual(inputReading.year);
     expect(resultingReading.quantity).toEqual(inputReading.quantity);
   });
+
+  it('should enrich the reading with the base charge', () => {
+    const resultingReading = enrichReading(inputReading);
+    expect(typeof resultingReading.baseCharge).toBe('number');
+  });
 });
```

And update `client3.js` to use the `baseCharge` field from enriched `reading` object:

```diff
diff --git a/src/client3.js b/src/client3.js
@@ -3,7 +3,7 @@ const { baseRate } = require('./tax-utils');

 const rawReading = acquireReading();
 const aReading = enrichReading(rawReading);
-const basicChargeAmount = calculateBaseCharge(aReading);
+const basicChargeAmount = aReading.baseCharge;

 console.log(`basic charge amount is ${basicChargeAmount}`);
```

With this done, we can remove the now unused `calculateBaseCharge` function:

```diff
diff --git a/src/client3.js b/src/client3.js
@@ -1,12 +1,7 @@
 const { acquireReading, enrichReading } = require('./reading');
-const { baseRate } = require('./tax-utils');

 const rawReading = acquireReading();
 const aReading = enrichReading(rawReading);
 const basicChargeAmount = aReading.baseCharge;

 console.log(`basic charge amount is ${basicChargeAmount}`);
-
-function calculateBaseCharge(aReading) {
-  return baseRate(aReading.month, aReading.year) * aReading.quantity;
-}
```

Moving forward, we can update `client1.js` to use the `baseCharge` field from the enriched `reading` object:

```diff
diff --git a/src/client1.js b/src/client1.js
@@ -1,7 +1,7 @@
-const { acquireReading } = require('./reading');
-const { baseRate } = require('./tax-utils');
+const { acquireReading, enrichReading } = require('./reading');

-const aReading = acquireReading();
-const baseCharge = baseRate(aReading.month, aReading.year) * aReading.quantity;
+const rawReading = acquireReading();
+const aReading = enrichReading(rawReading);
+const baseCharge = aReading.baseCharge;

 console.log(`base charge is ${baseCharge}`);
```

And also update `client2.js` to do the same:

```diff
diff --git a/src/client2.js b/src/client2.js
@@ -1,8 +1,9 @@
-const { acquireReading } = require('./reading');
-const { baseRate, taxThreshold } = require('./tax-utils');
+const { acquireReading, enrichReading } = require('./reading');
+const { taxThreshold } = require('./tax-utils');

-const aReading = acquireReading();
-const base = baseRate(aReading.month, aReading.year) * aReading.quantity;
+const rawReading = acquireReading();
+const aReading = enrichReading(rawReading);
+const base = aReading.baseCharge;
 const taxableCharge = Math.max(0, base - taxThreshold(aReading.year));

 console.log(`base charge is ${base}`);
```

Now we're just left with the calculation of the taxable charge to deal with. We can start by enriching the `reading` object with the `taxableCharge` calculation info as part of the `enrichReading` function:

```diff
diff --git a/src/reading/index.js b/src/reading/index.js
@@ -1,5 +1,5 @@
 const { cloneDeep } = require('lodash');
-const { baseRate } = require('../tax-utils');
+const { baseRate, taxThreshold } = require('../tax-utils');

 const reading = { customer: 'Ivan', quantity: 10, month: 5, year: 2017 };

@@ -10,6 +10,7 @@ function acquireReading() {
 function enrichReading(aReading) {
   const result = cloneDeep(aReading);
   result.baseCharge = calculateBaseCharge(result);
+  result.taxableCharge = Math.max(0, result.baseCharge - taxThreshold(aReading.year));
   return result;
 }

diff --git a/src/reading/index.test.js b/src/reading/index.test.js
@@ -15,4 +15,9 @@ describe('enrichReading', () => {
     const resultingReading = enrichReading(inputReading);
     expect(typeof resultingReading.baseCharge).toBe('number');
   });
+
+  it('should enrich the reading with the taxable charge', () => {
+    const resultingReading = enrichReading(inputReading);
+    expect(typeof resultingReading.taxableCharge).toBe('number');
+  });
 });
```

And then we can update `client2.js` to use the `taxableCharge` field from the enriched `reading` object:

```diff
diff --git a/src/client2.js b/src/client2.js
@@ -1,10 +1,9 @@
 const { acquireReading, enrichReading } = require('./reading');
-const { taxThreshold } = require('./tax-utils');

 const rawReading = acquireReading();
 const aReading = enrichReading(rawReading);
 const base = aReading.baseCharge;
-const taxableCharge = Math.max(0, base - taxThreshold(aReading.year));
+const taxableCharge = aReading.taxableCharge;

 console.log(`base charge is ${base}`);
 console.log(`taxable charge is ${taxableCharge}`);
```

And that's it! Now we have a transform function that enriches the `reading` object with the fields we need. This function can be reused anywhere in the system without worrying about specific formulas or details.

**Commit history**

The commit history for the steps detailed above is listed below:

| Commit SHA                                                                                                                              | Message                                                                     |
| --------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| [9699b10](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commit/9699b10e4498fd56cf59524e90f4d85505d9ce76) | introduce enrichReading function                                            |
| [87c373f](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commit/87c373fb819d84a015ec381b4de0f06087b50875) | update client3 to use enrichReading                                         |
| [45c1c8e](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commit/45c1c8ecb442cc6467b61e9aec3c71799a184636) | enrich reading with baseCharge in the enrichReading function                |
| [724f88a](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commit/724f88a3d345dedfc908c0e6f078760e7809576e) | update client3 to use baseCharge field from enriched reading obj            |
| [d152883](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commit/d15288360e7b67208fe6171bc06b1cc91a02b0d1) | remove unused calculateBaseCharge function                                  |
| [d3a7250](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commit/d3a7250d0becef7f1d566f93e1113f089d84e51c) | update client1 to use baseCharge from the enriched reading                  |
| [0871d35](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commit/0871d35bea4cfbb65db4bec24da9a3e06ef335ed) | update client2 to use the baseCharge field from the enriched reading obj    |
| [7a8b88f](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commit/7a8b88fa8bbb61b6638690f50cff941d1a3e0cd0) | enrich reading with taxableCharge info at enrichReading fn                  |
| [5ee217b](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commit/5ee217be0758cd6ec9cd1f2f78b24c2935df96b2) | update client2 to use the taxableCharge field from the enriched reading obj |

For the full commit history of the project, check the [Commit History tab](https://github.com/kaiosilveira/combine-functions-into-transform-refactoring/commits/5ee217be0758cd6ec9cd1f2f78b24c2935df96b2).
