# How to test?



- All of the tests and related files are in `pintos/src/tests`.


- Running "test_A" writes its

    - output to `test_A.out`.
    - verdict to `test_A.result`.



- To run single test, 
  
    - make the test_A.result explicitly.
    - e.g `make tests/threads/alarm-multiple.result`.


- use `VERBOSE=1` on make command to observe the progress of each test

    - e.g. make check VERBOSE=1



