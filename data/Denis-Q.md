https://github.com/code-423n4/2024-10-ramses-exchange/blob/main/test/v3/SPL.spec.ts
```
expect(secondsInRangeX96).approximately(
   (604799n * 2n ** 96n) / 2n,
   1n,
); 
```

AssertionError: expected 23958517116151021423950097037656063 to be close to 23958556730232278556118893809631232 +/- 1