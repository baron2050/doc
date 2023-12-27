---
title: 一个java状态机样例的代码
---

# 一个java状态机样例的代码

 在UML当中有状态机视图，这个状态机可以用于自动售货机，自动售票机等等场景，下面是用[Java](http://lib.csdn.net/base/java)代码模拟的一个状态机：

## 1.状态机接口



**[java]** [view plain](http://blog.csdn.net/seacean2000/article/details/10528153#) [copy](http://blog.csdn.net/seacean2000/article/details/10528153#) [print](http://blog.csdn.net/seacean2000/article/details/10528153#)[?](http://blog.csdn.net/seacean2000/article/details/10528153#)

1. **package** stateMachine; 
2. /** 
3.  \* 状态机接口 
4.  \* @author seacean 
5.  \* @date 2013-8-29 
6.  */ 
7. **public** **interface** State { 
8.   /** 
9.    \* 投入硬币 
10.    */ 
11.   **void** insertQuarter(); 
12.   /** 
13.    \* 根据摇动情况，处理摇动结果，返回处理结果，释放糖果 
14.    */ 
15.   **void** ejectQuarter(); 
16.   /** 
17.    \* 转动摇柄 
18.    */ 
19.   **void** turnCrank(); 
20.   /** 
21.    \* 机器放出糖果，处理机器内部状态，返回初始可投币状态 
22.    */ 
23.   **void** dispense(); 
24. } 





## 2.带有状态机的机器





**[java]** [view plain](http://blog.csdn.net/seacean2000/article/details/10528153#) [copy](http://blog.csdn.net/seacean2000/article/details/10528153#) [print](http://blog.csdn.net/seacean2000/article/details/10528153#)[?](http://blog.csdn.net/seacean2000/article/details/10528153#)

1. **package** stateMachine; 
2. /** 
3.  \* 机器类，包含多种状态，处理流程 
4.  \* @author seacean 
5.  \* @date 2013-8-29 
6.  */ 
7. **public** **class** Machine { 
8.   //机器本身包含所有的状态机 
9.   **private** State soldOutState; 
10.   **private** State noQuarterState; 
11.   **private** State hasQuarterState; 
12.   **private** State soldState; 
13.  
14.   **private** State state; //机器的当前状态 
15.   **private** **int** count = 0;//机器中当前糖果的数量 
16.   /** 
17.    \* 初始化机器，引入所有的状态机，初始化糖果数量，初始化机器状态 
18.    \* @param count 
19.    */ 
20.   **public** Machine(**int** count) { 
21. ​    **this**.soldOutState = **new** SoldOutState(**this**); 
22. ​    **this**.noQuarterState = **new** NoQuarterState(**this**); 
23. ​    **this**.hasQuarterState = **new** HasQuarterState(**this**); 
24. ​    **this**.soldState = **new** SoldState(**this**); 
25. ​    **this**.count = count; 
26. ​    **if** (**this**.count > 0) { 
27. ​      **this**.state = noQuarterState; 
28. ​    } 
29.   } 
30.   /** 
31.    \* 释放糖果时的内部处理程序 
32.    */ 
33.   **public** **void** releaseBall() { 
34. ​    System.out.println("a gumball comes rolling out the solt..."); 
35. ​    **if** (count > 0) { 
36. ​      count -= 1; 
37. ​    } 
38.   } 
39.    
40.   **public** **void** insertQuerter() { 
41. ​    state.insertQuarter();//加入硬币 
42.   } 
43.  
44.   **public** **void** ejectQuarter() { 
45. ​    state.ejectQuarter(); 
46.   } 
47.  
48.   **public** **void** turnCrank() { 
49. ​    state.turnCrank(); 
50. ​    state.dispense(); 
51.   } 
52.  
53.   **public** State getSoldOutState() { 
54. ​    **return** soldOutState; 
55.   } 
56.  
57.   **public** State getNoQuarterState() { 
58. ​    **return** noQuarterState; 
59.   } 
60.  
61.   **public** State getHasQuarterState() { 
62. ​    **return** hasQuarterState; 
63.   } 
64.  
65.   **public** State getSoldState() { 
66. ​    **return** soldState; 
67.   } 
68.  
69.   **public** State getState() { 
70. ​    **return** state; 
71.   } 
72.  
73.   **public** **int** getCount() { 
74. ​    **return** count; 
75.   } 
76.  
77.   **public** **void** setState(State state) { 
78. ​    **this**.state = state; 
79.   } 
80. } 



## 3.下面是状态机的一些实现类





**[java]** [view plain](http://blog.csdn.net/seacean2000/article/details/10528153#) [copy](http://blog.csdn.net/seacean2000/article/details/10528153#) [print](http://blog.csdn.net/seacean2000/article/details/10528153#)[?](http://blog.csdn.net/seacean2000/article/details/10528153#)

1. **package** stateMachine; 
2. /** 
3.  \* 机器处于没有投硬币的状态 
4.  \* @author seacean 
5.  \* @date 2013-8-29 
6.  */ 
7. **public** **class** NoQuarterState **implements** State { 
8.   **private** Machine machine; 
9.  
10.   **public** NoQuarterState(Machine machine) { 
11. ​    **this**.machine = machine; 
12.   } 
13.  
14.   @Override 
15.   **public** **void** insertQuarter() { 
16. ​    System.out.println("please insert a quarter!"); 
17. ​    machine.setState(machine.getHasQuarterState()); 
18.   } 
19.  
20.   @Override 
21.   **public** **void** ejectQuarter() { 
22. ​    System.out.println("please insert a quarter!"); 
23.   } 
24.  
25.   @Override 
26.   **public** **void** turnCrank() { 
27. ​    System.out.println("please insert a quarter!"); 
28.   } 
29.  
30.   @Override 
31.   **public** **void** dispense() { 
32. ​    System.out.println("please insert a quarter!"); 
33.   } 
34.  
35. } 

**[java]** [view plain](http://blog.csdn.net/seacean2000/article/details/10528153#) [copy](http://blog.csdn.net/seacean2000/article/details/10528153#) [print](http://blog.csdn.net/seacean2000/article/details/10528153#)[?](http://blog.csdn.net/seacean2000/article/details/10528153#)

1. **package** stateMachine; 
2. /** 
3.  \* 机器处于有硬币，有糖果，没有摇动的状态 
4.  \* @author seacean 
5.  \* @date 2013-8-29 
6.  */ 
7. **public** **class** HasQuarterState **implements** State { 
8.   **private** Machine machine; 
9.    
10.   **public** HasQuarterState(Machine machine){ 
11. ​    **this**.machine=machine; 
12.   } 
13.   @Override 
14.   **public** **void** insertQuarter() { 
15. ​    System.out.println("You can not insert another quarter!"); 
16.   } 
17.  
18.   @Override 
19.   **public** **void** ejectQuarter() { 
20. ​    System.out.println("Quarter returned!"); 
21. ​    machine.setState(machine.getNoQuarterState()); 
22.   } 
23.  
24.   @Override 
25.   **public** **void** turnCrank() { 
26. ​    System.out.println("You turned ... "); 
27. ​    machine.setState(machine.getSoldState()); 
28.   } 
29.  
30.   @Override 
31.   **public** **void** dispense() { 
32. ​    System.out.println("No gumball dispensed!"); 
33.   } 
34.  
35. } 



**[java]** [view plain](http://blog.csdn.net/seacean2000/article/details/10528153#) [copy](http://blog.csdn.net/seacean2000/article/details/10528153#) [print](http://blog.csdn.net/seacean2000/article/details/10528153#)[?](http://blog.csdn.net/seacean2000/article/details/10528153#)

1. **package** stateMachine; 
2.  
3. /** 
4.  \* 机器正在出售糖果的状态 
5.  \* 
6.  \* @author seacean 
7.  \* @date 2013-8-29 
8.  */ 
9. **public** **class** SoldState **implements** State { 
10.   **private** Machine machine; 
11.  
12.   **public** SoldState(Machine machine) { 
13. ​    **this**.machine = machine; 
14.   } 
15.  
16.   @Override 
17.   **public** **void** insertQuarter() { 
18. ​    System.out.println("please wait,we are already giving you a gumball!"); 
19.   } 
20.  
21.   @Override 
22.   **public** **void** ejectQuarter() { 
23. ​    System.out.println("Sorry, you have turned the crank!"); 
24.   } 
25.  
26.   @Override 
27.   **public** **void** turnCrank() { 
28. ​    System.out.println("Turning twice does not get you another gumball!"); 
29.   } 
30.  
31.   @Override 
32.   **public** **void** dispense() { 
33. ​    machine.releaseBall(); 
34. ​    **if** (machine.getCount() > 0) { 
35. ​      machine.setState(machine.getNoQuarterState()); 
36. ​    } **else** { 
37. ​      System.out.println("Out of Gumballs!"); 
38. ​      machine.setState(machine.getSoldOutState()); 
39. ​    } 
40.   } 
41.  
42. } 



**[java]** [view plain](http://blog.csdn.net/seacean2000/article/details/10528153#) [copy](http://blog.csdn.net/seacean2000/article/details/10528153#) [print](http://blog.csdn.net/seacean2000/article/details/10528153#)[?](http://blog.csdn.net/seacean2000/article/details/10528153#)

1. **package** stateMachine; 
2. /** 
3.  \* 机器处于无糖果状态 
4.  \* @author seacean 
5.  \* @date 2013-8-29 
6.  */ 
7. **public** **class** SoldOutState **implements** State { 
8.   **private** Machine machine; 
9.   **public** SoldOutState(Machine machine) { 
10. ​    **this**.machine=machine; 
11.   } 
12.  
13.   @Override 
14.   **public** **void** insertQuarter() { 
15. ​    System.out.println("Sorry, there is no gumball in the machine!"); 
16.   } 
17.  
18.   @Override 
19.   **public** **void** ejectQuarter() { 
20. ​    System.out.println("Sorry, there is no gumball in sold!"); 
21.   } 
22.  
23.   @Override 
24.   **public** **void** turnCrank() { 
25. ​    System.out.println("Sorry, there is no gumball!Turn is no meaning."); 
26. ​    machine.setState(machine.getNoQuarterState()); 
27.   } 
28.  
29.   @Override 
30.   **public** **void** dispense() { 
31. ​    System.out.println("Sorry, there is no gumball!"); 
32.   } 
33.  
34. } 





## 4.下面是测试类

**[java]** [view plain](http://blog.csdn.net/seacean2000/article/details/10528153#) [copy](http://blog.csdn.net/seacean2000/article/details/10528153#) [print](http://blog.csdn.net/seacean2000/article/details/10528153#)[?](http://blog.csdn.net/seacean2000/article/details/10528153#)

1. **package** stateMachine; 
2. //测试类 
3. **public** **class** StateMachineTest { 
4.   **public** **static** **void** main(String[] args) { 
5. ​     Machine machine=**new** Machine(10); 
6. ​     **for**(**int** i=0;i<11;i++){ 
7. ​     System.out.println(machine); 
8. ​     machine.insertQuerter(); 
9. ​     machine.turnCrank(); 
10. ​     } 
11.   } 
12. } 