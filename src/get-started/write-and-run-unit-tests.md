# Write and Run Unit Tests

## Add unit test cases in TiKV
Now let's take a look at how to add unit tests to tikv
If your modified / added code does not have a mod specifically for testing, 
you need to create a new test module first.  
If you already have one, skip this step.

### Add the tests module and #[cfg(test)]
First, add new module for unit test if the test module is not exists,
and add ```#[cfg(test)]``` macros.

#### What is #[cfg(test)]
The ```#[cfg(test)]``` annotation on the tests module tells Rust to compile and run the test code only when you run cargo test, 
not when you run cargo build. This saves compile time when you only want to build the library and saves space in the resulting compiled artifact because the tests are not included. 
You’ll see that because integration tests go in a different directory, 
they don’t need the #[cfg(test)] annotation. 
However, because unit tests go in the same files as the code, 
you’ll use ```#[cfg(test)]``` to specify that they shouldn’t be included in the compiled result.


```rust
#[cfg(test)]
mod tests{
    
   // write your test cases 
    
}

```  

## Write your test case:  
Add new funcitons for your test logic,add ```#[test]``` before your functions.
Use assert macros to check whether the results are consistent with your expectations.


there are 3 types macro of assert macros:
1. [assert](https://doc.rust-lang.org/std/macro.assert.html)  
Asserts that a boolean expression is true at runtime.

2. [assert_eq](https://doc.rust-lang.org/std/macro.assert_eq.html)  
Asserts that two expressions are equal to each other

3. [assert_ne](https://doc.rust-lang.org/std/macro.assert_ne.html)  
Asserts that two expressions are not equal to each other



```rust
#[cfg(test)]
mod tests{
    #[test]
    fn test_example(){

        let a=true;
        assert!(a);

        let x=1;
        let y=1;

        assert_eq!(x,y);

        let num01=1;
        let num02 =2;
        assert_ne!(1,2);

    }
}

```

## Run tests
Under source code directory of TiKV, when you're ready to test out your changes, use the `dev` task. 
It will format your codebase, build with `clippy` enabled, and run tests. 
This should run without failure before you create a PR. Unfortunately, some tests will fail intermittently and others don't pass on all platforms. 
If you're unsure, just ask!

```bash
make dev
```

You can run the test suite alone, or just run a specific test:

```bash
# Run the full suite
make test
# Run a specific test
./scripts/test $TESTNAME -- --nocapture
# Or using make
env EXTRA_CARGO_ARGS=$TESTNAME make test
```

## More information for unit tests in Rust
More information for unit tests in Rust, you can see:  
https://doc.rust-lang.org/rust-by-example/testing/unit_testing.html   
https://doc.rust-lang.org/book/ch11-03-test-organization.html