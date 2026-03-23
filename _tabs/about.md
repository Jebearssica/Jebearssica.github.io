---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

<center>
    <h1>吴联雄</h1>
    <div>
        <span>
            <img src="../assets/phone-solid.svg" width="18px">
            *******1986
        </span>
        ·
        <span>
            <img src="../assets/envelope-solid.svg" width="18px">
            jebearssica@outlook.com
        </span>
        ·
        <span>
            <img src="../assets/github-brands.svg" width="18px">
            <a href="https://github.com/jebearssica">jebearssica</a>
        </span>
        ·
        <span>
            <img src="../assets/rss-solid.svg" width="18px">
            <a href="https://jebearssica.github.io/">My Blog</a>
        </span>
    </div>
</center>

## <img src="../assets/info-circle-solid.svg" width="30px"> 个人信息

- 男, 1998 年出生
- C++ 研发工程师

## <img src="../assets/graduation-cap-solid.svg" width="30px"> 教育经历

- 硕士, 江苏大学, 计算机科学与技术专业, 2020.9~2023.6
- 学士, 江苏大学, 物联网专业, 2016.9~2020.6
- SCI 二区论文: Y. Yang, **L. Wu**, L. Zeng, T. Yan and Y. Zhan, "Joint Upsampling for Refocusing Light Fields Derived With Hybrid Lenses," in IEEE Transactions on Instrumentation and Measurement. (IF 5.6)
- CCF C 类会议: L. Zeng, **L. Wu**, Y. Yang, X. Shen, Y. Zhan, "Deep Weighted Guided Upsampling Network for Depth of Field Image Upsampling," in Proceedings of the 4th ACM International Conference on Multimedia in Asia, MMAsia 2022.

## <img src="../assets/briefcase-solid.svg" width="30px"> 工作经历

- **华芯巨数, C++ 研发工程师, 2022.8~至今**

负责 opc drc 开发与性能优化(啊, 没错, 最开始实习了十个月)

## <img src="../assets/project-diagram-solid.svg" width="30px"> 项目经历

- **HDRC 搭建, 2022.8~2023.3**

*C++11, computational geometry*

使得原有只支持 flat 的 opc drc 系统支持 hierarchy. 我在该项目中负责开发所有的基于 polygon 的 hierarchy 命令以及理解并总结 hierarchy 相关的 db 代码及原理.

- **HDRC 性能优化项目, 2023.3~至今**

*C++11, computational geometry, multithreading, optimization*

从底层算法, 版图优化, 多线程分布式支持, redis 启用等多方面, 对系统层面的时间/内存/disk开销等性能方面进行优化. 我在该项目中负责优化底层 boolean 算法, 负责系统输入端的版图优化, 负责部分命令的多线程支持.

对于底层算法, 通过 profiler 找到显著瓶颈部分, 修改 polygon 生成过程中的性能瓶颈; 对于输入版图优化, 通过 benchmark 的黑盒实验, 总结 benchmark 的 hierarchy tree 优化策略, 并通过特例触发验证猜想后实现自己的优化策略; 对于 merge 的多线程支持, 通过 per-level 拆分并查集算法尽可能实现更多流程的同步执行;

优化后的命令, 时间性能通常为 benchmark 的 15%~50% 避免了数量级上的时间性能差异. 对于优化后的版图, 在极端特例下的时间瓶颈消失. 对于多线程支持的命令, 线性度由于数据冲突的程度不同, 在 20%~70% 之间.

## <img src="../assets/tools-solid.svg" width="30px"> 技能清单

- C++/Python
- Computational Geometry(Planar Scanline, Convex Hull)
- Basic Algorithm(Union-Find, DP, Graph Traversal...)
- Git
