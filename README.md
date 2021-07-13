# Knowledge Share

This a collection of useful helper functions, industry knowledge and best practice on how to use the Betfair APIs collated by the ANZ automated community. 

Note: if you are keen to contribute to the repo please make a pull request or reach out to [automation@betfair.com.au](mailto:automation@betfair.com.au)


---
## Helper functions

### Splitting gallops & harness markets
Unfortunately there is no clean way to separate out gallops and harness markets via the Exchange APIs for ANZ races, expect by matching on the market name and searching for `pace` and `trot`.

An example of a function to return the market type:

``` typescript
// return true if name includes trot or pace, otherwise false
function isHarness(name: string): boolean {
    name = name.toLowerCase();
    return name.indexOf(" trot") !== -1 || name.indexOf(" pace") !== -1;
}
```

### Race type
The same logic as above applies to identifying race type.

``` typescript
// return true if name includes Mdn otherwise false
function isMaiden(name: string): boolean {
    name = name.toLowerCase();
    return name.indexOf(" mdn") !== -1;
}
```

### Race length 
Equally, this is how you can identify race length.

``` typescript
// takes in market name and returns race length 
function raceLen(name: string): number {
    // # return race length
    // # input samples: 
    // # 'R6 1400m Grp1' -> ('R6','1400m','grp1')
    // # 'R1 1609m Trot M' -> ('R1', '1609m', 'trot')
    // # 'R4 1660m Pace M' -> ('R4', '1660m', 'pace')
    var parts = name.split(' ');
    var race_len = parts[1];
    var x = race_len.split('m');
    var y: number = +x[0];

    console.log("raceLen", name, x, y)

    return y;
}
```

---
## Pricing logic

### Available to back order
Often you want to look at the 'market favourite' in a race, which can be defined as the runner with the shorted best available to back price.

``` typescript
// returns sorted order of selections based on best atb price
function AtbOrder(sels: Selection[]): number[]  {
    const sorted = [ ...sels ].sort( (a, b) => {
        if (a.status == SelStatus.Active && b.status != SelStatus.Active) {
            return -1;
        } else if (a.status != SelStatus.Active && b.status == SelStatus.Active) {
            return 1;
        } else {
            const ai = a.lay[0]?.price, bi = b.lay[0]?.price;
            return (ai ?? 1000) - (bi ?? 1000);
        }
    });
    const out = sels.map( s1 => sorted.findIndex( s2 => s1.id == s2.id ));
    return out;
}
```

### Weighted average price (WAP)
Weighted average price is a good way of removing outlier prices and looking at the market more holistically.

#### Best available to lay WAP

``` typescript
// calculating wap across top three available to lay boxes
function atlWap(atl0, atl1, atl2): number {
    const vol = atl0?.volume + atl1?.volume + atl2?.volume;
    const priceXvol = (atl0?.volume * atl0?.price) + (atl1?.volume * atl1?.price) + (atl2?.volume * atl2?.price)

    return priceXvol/vol;
}
```

#### Best available to back WAP

``` typescript
// calculating wap across top three available to back boxes
function atbWap(atb0, atb1, atb2): number {
    const vol = atb0?.volume + atb1?.volume + atb2?.volume;
    const priceXvol = (atb0?.volume * atb0?.price) + (atb1?.volume * atb1?.price) + (atb2?.volume * atb2?.price)

    return priceXvol/vol;
}
```

--- 
## Staking logic

### Kelly staking
This is an approach to creating Kelly staking functions for back and lay stakes. 

#### Back Kelly

``` typescript
/**
 * Kelly staking backing
 * http://www.aussportsbetting.com/2010/07/18/kelly-criterion-backing-andlaying-bets-with-betfair/
 * f = p(d-1)(1-c)-q 
        /(d-1)(1-c)
 * f = the fraction of your bankroll (account balance) to bet on a particular outcome of a sporting event
 * d = decimal betting odds for that outcome 
 * p = probability of success (1/rating) 
 * q = probability of failure (1-P)
 * c = commission rate
 * kellyApp: 1 = full, 2 = half, 4 = quarter
 **/
function kellyMultBack(price: number, rating: number, kellyApp: number = 1, commRate: number = 0): number {
    const d = price;
    const c = commRate;
    const p = 1 / rating;
    const q = 1 - p;

    return ((p * (d - 1) * (1 - c) - q) / ((d - 1) * (1 - c))) / kellyApp;
}
```

#### Lay Kelly

``` typescript
/**
 * Kelly staking laying
 * http://www.aussportsbetting.com/2010/07/18/kelly-criterion-backing-andlaying-bets-with-betfair/
 * f = (q*(1-c)-p*(d-1))
 *      /((d-1)*(1-c)
 * f = the fraction of your bankroll (account balance) to bet on a particular outcome of a sporting event
 * d = decimal betting odds for that outcome 
 * p = probability of success (1/rating) 
 * q = probability of failure (1-P)
 * c = commission rate
 * kellyApp: 1 = full, 2 = half, 4 = quarter
 **/
function kellyMultLay(price: number, rating: number, kellyApp: number = 1, commRate: number = 0): number {
    const d = price;
    const c = commRate;
    const p = 1 / rating;
    const q = 1 - p;

    return ((q * (1 - c) - p * (d - 1)) / ((d - 1) * (1 - c))) / kellyApp;
}
```

---
## Best practice

### Market start time
Racing markets notoriously go in play later, and/or their start time get pushed back without necessarily being updated on the Betfair API. 

One way of guarding against this is to include a back market percentage check, to use the wisdom of the crowd to inform your betting decisions. i.e. you might check that it's < 30 seconds to the scheduled market start time AND that the BMP < 103%.

BMP is calculated as the sum of (100 / best available to back odds) for all selections in the market.

In a two horse race where both selections were $2 it would be:
`(100/2) + (100/2) = 100%`

---
## Industry knowledge


---
### Disclaimer
These functions are provided for educational purposes only and with no warranty. If you choose to use any or all of the above contents you do so at your own risk. The authors accept no liability for any losses incurred as a result of individuals choosing to use the above content in their own betting strategies. Make sure you test all strategies thoroughly to ensure you're happy with them before betting with significant stake sizes or leaving a betting program unattended. If you think you might have a gambling problem please seek help. Gambling Help Online.

