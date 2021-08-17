UBO(Uniform Buffer Object)可用于在多个shader program之间共享uniform变量，批量管理。

SSBO(Shader Storage Buffer Object)，可用于不同shader之间共享数据以及缓存数据。

UBO与SSBO区别：

大小限制：UBO大小限制为KB级别，64KB或128KB，具体与硬件相关。而SSBO则灵活得多，大小可为显存大小（GB），因而更适合与存储大量的数据。而且SSBO可以在shader运行时再确定大小，而非编译时。
 	
存取速度：UBO存储区为显卡的常量区，而SSBO为全局的显存，UBO速度高于SSBO。
 	
读写：UBO对于shader来讲是只读常量，只能通过外部程序更新。SSBO则是shader可以读写的（例如在compute shader中更新定点的位置）。
 	
SSBO还有一个特点，就是任何其他类型的OpenGL buffer都可绑定到GL_SHADER_STORAGE_BUFFER 目标上，因而很灵活。