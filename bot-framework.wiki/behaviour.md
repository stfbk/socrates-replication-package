# Behaviour

There are **three classes** representing three kinds of behaviours:

- [Random Behaviour](#random-behaviour)
- [Boundary Behaviour](#boundary-behaviour)
- [Overflow Behaviour](#overflow-behaviour)

All the classes extend the abstract class *Behaviour* and implement the abstract method **performAction()**.

## Random Behaviour

RandomBehaviour's *performAction* randomly select a function to call, generates the corresponding parameters and execute the transaction. The core of the random behaviour is 
the helper class `RandomValueGenerator`.

`RandomValueGenerator` has a method *getRandomParameters* which takes in input an object of type *ABI.Types.Function* and, based on type constraints or manually defined constraints, it generates a sequence of parameters with pseudo-random values.

The following snippet of code represents a portion of the function *newRandomValueFromType*. This function creates and returns a compatible new value which respects some type constraints.

```javascript
    ...

// check if type is an "int"
if (paramType.includes("int")){

    // check unsigned
    let unsigned = paramType.startsWith('uint');

    // check type range
    let numbersInStringType = paramType.match(/\d+/g);
    let bits = numbersInStringType ? parseInt(numbersInStringType[0]) : 256;

    // define the range
    // use manual constraints if defined
    let minForType: IBigNumber = unsigned ? new BigNumber(0) : new BigNumber(2).pow(bits-1).times(-1).plus(1);
    let maxForType: IBigNumber = unsigned ? new BigNumber(2).pow(bits).minus(1) : new BigNumber(2).pow(bits-1).minus(2);
    minBigNumber = paramContraints.minValue ? new BigNumber(paramContraints.minValue) : minForType;
    maxBigNumber = paramContraints.maxValue ? new BigNumber(paramContraints.maxValue) : maxForType;

    // get a random number in that range
    return this.getRandomBigNumber(minBigNumber, maxBigNumber);
}

// Handle other types
switch (paramType) {
    case 'address':
        let randomAccount = RandomValueGenerator.getRandomElement<Account>(botAccounts);
        return randomAccount.address;
    case 'bool':
        // return a default boolean or a random (50:50) one
        let defaultBoolean = paramContraints.defaultValue;
        return defaultBoolean ? defaultBoolean : Math.random() >= 0.5;
    case 'string':
        randomSize = ... // random size from user defined constraints
        return this.getRandomString(randomSize);
    default:
        throw new Error("Unhandled Solidity Type");
}
```


## Boundary Behaviour

The Boundary behaviour extends the *RandomBehaviour* by substituting the *RandomValueGenerator* instance with an instance of class `BoundaryValueGenerator`.

The *BoundaryValueGenenator* redefines the selection strategy of the integers. Given a range delimited by two values *min* and *max*, it selects a new integer value with respect to the following selection strategy:

```typescript
/**
    * 1/10 it returns the min
    * 1/10 it returns the max
    * 1/10 it returns a number close to min
    * 1/10 it returns a number close to max
    * 6/10 it returns a random number [0, delta]
    * @returns {BigNumber} random BigNumber value
*/
protected getBoundaryBigNumber(min: IBigNumber, max: IBigNumber) : any {
    let choice = RandomValueGenerator.getRandomInt(0, 10); // min included, max not
    if (choice == 0) return min;
    if (choice == 1) return max;
    if (choice == 2) return this.getRandomBigNumber(min, min.plus(this._delta));
    if (choice == 3) return this.getRandomBigNumber(max.minus(this._delta), max);
    return new BigNumber(RandomValueGenerator.getRandomInt(0, this._delta));
}
```

## Overflow behaviour

The `Overflow behaviour` requires the scripts generated by the **overflow-script-generator**. The function *performAction* does the following:

1. Selects the **function to call** choosing randomly among the functions which have an *overflow script* generated.
2. **Executes** its overflow script (call `callZ3Solver`)
3. Parses the execution output **retrieving** the parameters (call `getFunctionParams`)
4. **Submits** a new transaction to the function using those parameters

In detail:

```javascript
public async performAction(): Promise<any> {
    let contract: Contract = this._target as Contract;
    let abi: ABI.Types.NameToFunctionMap = this._target.abi;

    // Get all the callable functions
    let listOfFiles: [string] = await glob('./generated/*_z3.py');
    let callableFunctions: ABI.Types.Function[] = new Array<ABI.Types.Function>();
    for (const [ key, value ] of Object.entries(abi)) {
        let expectedPath = "./generated/" + key + "_z3.py";
        if (['view', 'pure'].includes(value['stateMutability']) == false && listOfFiles.includes(expectedPath)) {
            callableFunctions.push(value);
        }
    }

    if (callableFunctions.length == 0) {
        throw new Error("No callable function");
    }

    // Choose a random function among the ones available
    let chosenAction = RandomValueGenerator.getRandomElement<ABI.Types.Function>(callableFunctions);

    // Random distinct option: force the elements in array to be distinct
    let distinctOption: number = RandomValueGenerator.getRandomElement([0, 1]);

    // Exec the script
    let results = await this.callZ3Solver(chosenAction.name, this._target, this._target.artifactsPath, distinctOption);

    if (results && results.sat){
        // parse the outputs and get concrete parameters
        let functionParams = this.getFunctionParams(chosenAction, results);

        // submit new transaction
        let actionResult = await this.execTransaction(contract, chosenAction, functionParams);

        if (actionResult.successful) {
            return actionResult; // successful: true, overflow: true
        }
    }

    // if here returns result for an unsuccessful transaction
    return  {"label": "action", "successful": false,
    "overflow": false, address: this._account.address,
    "action": chosenAction.name};
}
```

The function **getFunctionParams** parses the overflow script output (JSON) and, for each parameter it calls the method *getConcreteValue* to convert back the values from the Z3 representation.
For example, the scripts represent *addresses* as indexes of the *ganache accounts list*.

```javascript
private getConcreteValue(z3value: string, type: string) : any | null {

    let isConsistent: boolean =  this.checkTypeConsistency(z3value, type);
    if (!isConsistent) return null;

    // value is consistent
    if (type.includes('address')){
        // Array of index of address accounts
        const addressId = parseInt(z3value);
        return this._context.accounts[addressId].address;
    } else {
        return new BigNumber(z3value);
    }
}
```