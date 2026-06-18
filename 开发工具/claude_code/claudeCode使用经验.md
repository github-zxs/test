#### 深度思考
1. 开启深度思考等级
	 - think   +
	 - think hard ++
	 - think harder  +++
	 - ultrathink ++++
2. 使用时机
     -  复杂架构决策
     -  深度分析才能解决时 
     - claude钻进死胡同不停打转
     - 需要先规划在动手的大任务
3.  使用方法
	 - **思考**: 要求claude先思考和研究(不要写代码)
	 - **规划**: 让它创建一个计划
	 - **确认**:确认计划看起来不错
	 - **执行**:要求它实施计划


#### output styles
**作用:** 用于控制其响应的**语气、结构、详细程度和交互方式**，本质是一套预设 / 自定义的系统提示（system prompt），可快速适配不同工作流
**原理:** 替换claude的核心指令

#### 启动参数
claude --dangerously-skip-permissions  --verbose