
## how to test?


```bash
  cd /workspace/threads/build/
  rm tests/threads/alarm-multiple.output
  make tests/threads/alarm-multiple.result
```

### test list

```
ls threads/build/tests/threads/ | grep result

alarm-multiple.result
alarm-negative.result
alarm-priority.result
alarm-simultaneous.result
alarm-single.result
alarm-zero.result
priority-change.result
priority-condvar.result
priority-donate-chain.result
priority-donate-lower.result
priority-donate-multiple2.result
priority-donate-multiple.result
priority-donate-nest.result
priority-donate-one.result
priority-donate-sema.result
priority-fifo.result
priority-preempt.result
priority-sema.result
```


## test history

