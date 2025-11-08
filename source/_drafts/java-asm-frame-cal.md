---
title: java-asm-frame-cal
tags:
cover:
---



# ASM 操作数栈大小计算



# 背景



最近在写一个ASM插件，逻辑很简单就是将new方法进行hook转接到一个static方法中去。

中途会修改操作数栈的大小。

在书写插件的过程逐渐在COMPUTE_FRAMES,COMPUTE_MAX,EXPAND_FRAMES参数中迷失。查阅资料良久也是没有任何的文章讲解堆栈的计算原理。

因此有了本文章。



# Frame创建



labels

frame





## visitJump



1.将jumpCode添加到code列表中

2.添加后继节点，分两种情况

​	a.非goto指令

​		i.将jump跳转节点添加为后继节点

​		ii.调用visitLabel开辟新的block节点

​	b. goto指令跳转

​		i.将jump跳转节点添加为后继节点

​		ii.清空当前节点

```java
// MethodWriter.java
public void visitJumpInsn(final int opcode, final Label label) {
    lastBytecodeOffset = code.length;
    // Add the instruction to the bytecode of the method.
    // Compute the 'base' opcode, i.e. GOTO or JSR if opcode is GOTO_W or JSR_W, otherwise opcode.
    // #1 将jumpCode添加到method code列表中
    int baseOpcode =
        opcode >= Constants.GOTO_W ? opcode - Constants.WIDE_JUMP_OPCODE_DELTA : opcode;
    boolean nextInsnIsJumpTarget = false;
    if ((label.flags & Label.FLAG_RESOLVED) != 0
        && label.bytecodeOffset - code.length < Short.MIN_VALUE) {
      // Case of a backward jump with an offset < -32768. In this case we automatically replace GOTO
      // with GOTO_W, JSR with JSR_W and IFxxx <l> with IFNOTxxx <L> GOTO_W <l> L:..., where
      // IFNOTxxx is the "opposite" opcode of IFxxx (e.g. IFNE for IFEQ) and where <L> designates
      // the instruction just after the GOTO_W.
      if (baseOpcode == Opcodes.GOTO) {
        code.putByte(Constants.GOTO_W);
      } else if (baseOpcode == Opcodes.JSR) {
        code.putByte(Constants.JSR_W);
      } else {
        // Put the "opposite" opcode of baseOpcode. This can be done by flipping the least
        // significant bit for IFNULL and IFNONNULL, and similarly for IFEQ ... IF_ACMPEQ (with a
        // pre and post offset by 1). The jump offset is 8 bytes (3 for IFNOTxxx, 5 for GOTO_W).
        code.putByte(baseOpcode >= Opcodes.IFNULL ? baseOpcode ^ 1 : ((baseOpcode + 1) ^ 1) - 1);
        code.putShort(8);
        // Here we could put a GOTO_W in theory, but if ASM specific instructions are used in this
        // method or another one, and if the class has frames, we will need to insert a frame after
        // this GOTO_W during the additional ClassReader -> ClassWriter round trip to remove the ASM
        // specific instructions. To not miss this additional frame, we need to use an ASM_GOTO_W
        // here, which has the unfortunate effect of forcing this additional round trip (which in
        // some case would not have been really necessary, but we can't know this at this point).
        code.putByte(Constants.ASM_GOTO_W);
        hasAsmInstructions = true;
        // The instruction after the GOTO_W becomes the target of the IFNOT instruction.
        nextInsnIsJumpTarget = true;
      }
      label.put(code, code.length - 1, true);
    } else if (baseOpcode != opcode) {
      // Case of a GOTO_W or JSR_W specified by the user (normally ClassReader when used to remove
      // ASM specific instructions). In this case we keep the original instruction.
      code.putByte(opcode);
      label.put(code, code.length - 1, true);
    } else {
      // Case of a jump with an offset >= -32768, or of a jump with an unknown offset. In these
      // cases we store the offset in 2 bytes (which will be increased via a ClassReader ->
      // ClassWriter round trip if it turns out that 2 bytes are not sufficient).
      code.putByte(baseOpcode);
      label.put(code, code.length - 1, false);
    }

    // If needed, update the maximum stack size and number of locals, and stack map frames.
    // #2 添加后继节点。
    if (currentBasicBlock != null) {
      Label nextBasicBlock = null;
      if (compute == COMPUTE_ALL_FRAMES) {
        currentBasicBlock.frame.execute(baseOpcode, 0, null, null);
        // Record the fact that 'label' is the target of a jump instruction.
        label.getCanonicalInstance().flags |= Label.FLAG_JUMP_TARGET;
        // Add 'label' as a successor of the current basic block.
        addSuccessorToCurrentBasicBlock(Edge.JUMP, label);
        if (baseOpcode != Opcodes.GOTO) {
          // The next instruction starts a new basic block (except for GOTO: by default the code
          // following a goto is unreachable - unless there is an explicit label for it - and we
          // should not compute stack frame types for its instructions).
          nextBasicBlock = new Label();
        }
      } else if (compute == COMPUTE_INSERTED_FRAMES) {
        currentBasicBlock.frame.execute(baseOpcode, 0, null, null);
      } else if (compute == COMPUTE_MAX_STACK_AND_LOCAL_FROM_FRAMES) {
        // No need to update maxRelativeStackSize (the stack size delta is always negative).
        relativeStackSize += STACK_SIZE_DELTA[baseOpcode];
      } else {
        if (baseOpcode == Opcodes.JSR) {
          // Record the fact that 'label' designates a subroutine, if not already done.
          if ((label.flags & Label.FLAG_SUBROUTINE_START) == 0) {
            label.flags |= Label.FLAG_SUBROUTINE_START;
            hasSubroutines = true;
          }
          currentBasicBlock.flags |= Label.FLAG_SUBROUTINE_CALLER;
          // Note that, by construction in this method, a block which calls a subroutine has at
          // least two successors in the control flow graph: the first one (added below) leads to
          // the instruction after the JSR, while the second one (added here) leads to the JSR
          // target. Note that the first successor is virtual (it does not correspond to a possible
          // execution path): it is only used to compute the successors of the basic blocks ending
          // with a ret, in {@link Label#addSubroutineRetSuccessors}.
          addSuccessorToCurrentBasicBlock(relativeStackSize + 1, label);
          // The instruction after the JSR starts a new basic block.
          nextBasicBlock = new Label();
        } else {
          // No need to update maxRelativeStackSize (the stack size delta is always negative).
          relativeStackSize += STACK_SIZE_DELTA[baseOpcode];
          addSuccessorToCurrentBasicBlock(relativeStackSize, label);
        }
      }
      // If the next instruction starts a new basic block, call visitLabel to add the label of this
      // instruction as a successor of the current block, and to start a new basic block.
      if (nextBasicBlock != null) {
        if (nextInsnIsJumpTarget) {
          nextBasicBlock.flags |= Label.FLAG_JUMP_TARGET;
        }
        visitLabel(nextBasicBlock);
      }
      if (baseOpcode == Opcodes.GOTO) {
        endCurrentBasicBlockWithNoSuccessor();
      }
    }
  }
```







## visitLabel



1.label重定向

2.添加后继节点

```java
public void visitLabel(final Label label) {
    // Resolve the forward references to this label, if any.
    hasAsmInstructions |= label.resolve(code.data, stackMapTableEntries, code.length);
    // visitLabel starts a new basic block (except for debug only labels), so we need to update the
    // previous and current block references and list of successors.
    if ((label.flags & Label.FLAG_DEBUG_ONLY) != 0) {
      return;
    }
    if (compute == COMPUTE_ALL_FRAMES) {
      if (currentBasicBlock != null) {
        if (label.bytecodeOffset == currentBasicBlock.bytecodeOffset) {
          // We use {@link Label#getCanonicalInstance} to store the state of a basic block in only
          // one place, but this does not work for labels which have not been visited yet.
          // Therefore, when we detect here two labels having the same bytecode offset, we need to
          // - consolidate the state scattered in these two instances into the canonical instance:
          currentBasicBlock.flags |= (label.flags & Label.FLAG_JUMP_TARGET);
          // - make sure the two instances share the same Frame instance (the implementation of
          // {@link Label#getCanonicalInstance} relies on this property; here label.frame should be
          // null):
          label.frame = currentBasicBlock.frame;
          // - and make sure to NOT assign 'label' into 'currentBasicBlock' or 'lastBasicBlock', so
          // that they still refer to the canonical instance for this bytecode offset.
          return;
        }
        // End the current basic block (with one new successor).
        addSuccessorToCurrentBasicBlock(Edge.JUMP, label);
      }
      // Append 'label' at the end of the basic block list.
      if (lastBasicBlock != null) {
        if (label.bytecodeOffset == lastBasicBlock.bytecodeOffset) {
          // Same comment as above.
          lastBasicBlock.flags |= (label.flags & Label.FLAG_JUMP_TARGET);
          // Here label.frame should be null.
          label.frame = lastBasicBlock.frame;
          currentBasicBlock = lastBasicBlock;
          return;
        }
        lastBasicBlock.nextBasicBlock = label;
      }
      lastBasicBlock = label;
      // Make it the new current basic block.
      currentBasicBlock = label;
      // Here label.frame should be null.
      label.frame = new Frame(label);
    } else if (compute == COMPUTE_INSERTED_FRAMES) {
      if (currentBasicBlock == null) {
        // This case should happen only once, for the visitLabel call in the constructor. Indeed, if
        // compute is equal to COMPUTE_INSERTED_FRAMES, currentBasicBlock stays unchanged.
        currentBasicBlock = label;
      } else {
        // Update the frame owner so that a correct frame offset is computed in Frame.accept().
        currentBasicBlock.frame.owner = label;
      }
    } else if (compute == COMPUTE_MAX_STACK_AND_LOCAL) {
      if (currentBasicBlock != null) {
        // End the current basic block (with one new successor).
        currentBasicBlock.outputStackMax = (short) maxRelativeStackSize;
        addSuccessorToCurrentBasicBlock(relativeStackSize, label);
      }
      // Start a new current basic block, and reset the current and maximum relative stack sizes.
      currentBasicBlock = label;
      relativeStackSize = 0;
      maxRelativeStackSize = 0;
      // Append the new basic block at the end of the basic block list.
      if (lastBasicBlock != null) {
        lastBasicBlock.nextBasicBlock = label;
      }
      lastBasicBlock = label;
    } else if (compute == COMPUTE_MAX_STACK_AND_LOCAL_FROM_FRAMES && currentBasicBlock == null) {
      // This case should happen only once, for the visitLabel call in the constructor. Indeed, if
      // compute is equal to COMPUTE_MAX_STACK_AND_LOCAL_FROM_FRAMES, currentBasicBlock stays
      // unchanged.
      currentBasicBlock = label;
    }
  }
```





# Frame计算 & 合并



其实本质上就是调用visitMaxs触发堆栈大小的计算

```java
public void visitMaxs(final int maxStack, final int maxLocals) {
    if (compute == COMPUTE_ALL_FRAMES) {
      computeAllFrames();
    } else if (compute == COMPUTE_MAX_STACK_AND_LOCAL) {
      computeMaxStackAndLocal();
    } else if (compute == COMPUTE_MAX_STACK_AND_LOCAL_FROM_FRAMES) {
      this.maxStack = maxRelativeStackSize;
    } else {
      this.maxStack = maxStack;
      this.maxLocals = maxLocals;
    }
  }
```



## compute_all_frames



核心对所有的frame进行遍历，不断往待处理队列中添加处理项。

1.处理handler的CFG

2.处理首个frame

3.使用固定点算法计算最后的堆栈大小

​	a. 从basic block中移除一个元素进行处理, 并更新最大堆栈大小

​	b. 尝试对所有的子节点进行合并

```java
private void computeAllFrames() {
    // Complete the control flow graph with exception handler blocks.
    Handler handler = firstHandler;
    while (handler != null) {
      String catchTypeDescriptor =
          handler.catchTypeDescriptor == null ? "java/lang/Throwable" : handler.catchTypeDescriptor;
      int catchType = Frame.getAbstractTypeFromInternalName(symbolTable, catchTypeDescriptor);
      // Mark handlerBlock as an exception handler.
      Label handlerBlock = handler.handlerPc.getCanonicalInstance();
      handlerBlock.flags |= Label.FLAG_JUMP_TARGET;
      // Add handlerBlock as a successor of all the basic blocks in the exception handler range.
      Label handlerRangeBlock = handler.startPc.getCanonicalInstance();
      Label handlerRangeEnd = handler.endPc.getCanonicalInstance();
      while (handlerRangeBlock != handlerRangeEnd) {
        handlerRangeBlock.outgoingEdges =
            new Edge(catchType, handlerBlock, handlerRangeBlock.outgoingEdges);
        handlerRangeBlock = handlerRangeBlock.nextBasicBlock;
      }
      handler = handler.nextHandler;
    }

    // Create and visit the first (implicit) frame.
    Frame firstFrame = firstBasicBlock.frame;
    firstFrame.setInputFrameFromDescriptor(symbolTable, accessFlags, descriptor, this.maxLocals);
    firstFrame.accept(this);

    // Fix point algorithm: add the first basic block to a list of blocks to process (i.e. blocks
    // whose stack map frame has changed) and, while there are blocks to process, remove one from
    // the list and update the stack map frames of its successor blocks in the control flow graph
    // (which might change them, in which case these blocks must be processed too, and are thus
    // added to the list of blocks to process). Also compute the maximum stack size of the method,
    // as a by-product.
    Label listOfBlocksToProcess = firstBasicBlock;
    listOfBlocksToProcess.nextListElement = Label.EMPTY_LIST;
    int maxStackSize = 0;
    while (listOfBlocksToProcess != Label.EMPTY_LIST) {
      // Remove a basic block from the list of blocks to process.
      Label basicBlock = listOfBlocksToProcess;
      listOfBlocksToProcess = listOfBlocksToProcess.nextListElement;
      basicBlock.nextListElement = null;
      // By definition, basicBlock is reachable.
      basicBlock.flags |= Label.FLAG_REACHABLE;
      // Update the (absolute) maximum stack size.
      int maxBlockStackSize = basicBlock.frame.getInputStackSize() + basicBlock.outputStackMax;
      if (maxBlockStackSize > maxStackSize) {
        maxStackSize = maxBlockStackSize;
      }
      // Update the successor blocks of basicBlock in the control flow graph.
      Edge outgoingEdge = basicBlock.outgoingEdges;
      while (outgoingEdge != null) {
        Label successorBlock = outgoingEdge.successor.getCanonicalInstance();
        boolean successorBlockChanged =
            basicBlock.frame.merge(symbolTable, successorBlock.frame, outgoingEdge.info);
        if (successorBlockChanged && successorBlock.nextListElement == null) {
          // If successorBlock has changed it must be processed. Thus, if it is not already in the
          // list of blocks to process, add it to this list.
          successorBlock.nextListElement = listOfBlocksToProcess;
          listOfBlocksToProcess = successorBlock;
        }
        outgoingEdge = outgoingEdge.nextEdge;
      }
    }

    // Loop over all the basic blocks and visit the stack map frames that must be stored in the
    // StackMapTable attribute. Also replace unreachable code with NOP* ATHROW, and remove it from
    // exception handler ranges.
    Label basicBlock = firstBasicBlock;
    while (basicBlock != null) {
      if ((basicBlock.flags & (Label.FLAG_JUMP_TARGET | Label.FLAG_REACHABLE))
          == (Label.FLAG_JUMP_TARGET | Label.FLAG_REACHABLE)) {
        basicBlock.frame.accept(this);
      }
      if ((basicBlock.flags & Label.FLAG_REACHABLE) == 0) {
        // Find the start and end bytecode offsets of this unreachable block.
        Label nextBasicBlock = basicBlock.nextBasicBlock;
        int startOffset = basicBlock.bytecodeOffset;
        int endOffset = (nextBasicBlock == null ? code.length : nextBasicBlock.bytecodeOffset) - 1;
        if (endOffset >= startOffset) {
          // Replace its instructions with NOP ... NOP ATHROW.
          for (int i = startOffset; i < endOffset; ++i) {
            code.data[i] = Opcodes.NOP;
          }
          code.data[endOffset] = (byte) Opcodes.ATHROW;
          // Emit a frame for this unreachable block, with no local and a Throwable on the stack
          // (so that the ATHROW could consume this Throwable if it were reachable).
          int frameIndex = visitFrameStart(startOffset, /* numLocal= */ 0, /* numStack= */ 1);
          currentFrame[frameIndex] =
              Frame.getAbstractTypeFromInternalName(symbolTable, "java/lang/Throwable");
          visitFrameEnd();
          // Remove this unreachable basic block from the exception handler ranges.
          firstHandler = Handler.removeRange(firstHandler, basicBlock, nextBasicBlock);
          // The maximum stack size is now at least one, because of the Throwable declared above.
          maxStackSize = Math.max(maxStackSize, 1);
        }
      }
      basicBlock = basicBlock.nextBasicBlock;
    }

    this.maxStack = maxStackSize;
  }
```





# 其他





## expand_frams



expand_frame会导致asm异常



## compute_frame



compute_frames获取class导致崩溃

