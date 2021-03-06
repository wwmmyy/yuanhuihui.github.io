Proc-todo
thread-todo
node-aync_todo

## Client

writeTransactionData //BC_TRANSACTION
waitForResponse
{
    while (1) {
        talkWithDriver()
        switch (cmd)
            case BR_TRANSACTION_COMPLETE:
            case BR_DEAD_REPLY: 
            case BR_FAILED_REPLY: 
            case BR_REPLY:  goto finish;
            default: executeCommand() //继续执行
    }
}

其中前4个BR_命令会退出waitForResponse()循环.

## Server

joinThreadPool
{
    while (1){
        processPendingDerefs() // BC_XXX
        getAndExecuteCommand()
            talkWithDriver()
            executeCommand()
    }
}

## 其中

    talkWithDriver
        binder_thread_write //BC_TRANSACTION
          //BINDER_WORK_TRANSACTION + BINDER_WORK_TRANSACTION_COMPLETE
          binder_transaction 
        binder_thread_read //生成BR_TRANSACTION_COMPLETE + BR_TRANSACTION
        
    executeCommand
        case BR_TRANSACTION:
            BBinder.transact(&reply)
            sendReply(reply)
              writeTransactionData()  //BC_REPLY
              waitForResponse()
            freeBuffer() //BC_FREE_BUFFER
    
## 进一步转换

waitForResponse
{
    while (1) {
        binder_thread_write //BC_TRANSACTION
        binder_thread_read //BR_TRANSACTION_COMPLETE + BR_TRANSACTION
        
        switch (cmd)
            case BR_REPLY:  goto finish;            
            default: executeCommand() 
                BBinder.transact(&reply)
                sendReply(reply)
                  writeTransactionData()  //发送BC_REPLY
                  waitForResponse() //等待BR_TRANSACTION_COMPLETE
                freeBuffer() //BC_FREE_BUFFER
    }
}  

joinThreadPool
{
    while (1){
        processPendingDerefs() 
            writeTransactionData // BC_TRANSACTION
            waitForResponse //等待BR_REPLY(非oneway)

        binder_thread_write  // 写入BC_XXX
        binder_thread_read  // 读取BR_TRANSACTION
        
        executeCommand
            BBinder.transact(&reply)
            sendReply(reply)
              writeTransactionData()  //写入BC_REPLY
              waitForResponse()  //等待BR_TRANSACTION_COMPLETE
            freeBuffer() //发送BC_FREE_BUFFER, async_todo加入当前thread-todo
    }
}
