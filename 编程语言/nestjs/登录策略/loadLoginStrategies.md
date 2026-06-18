import * as glob from "glob";

import { join } from 'path';

import { LoginStrategy } from "../login/strategies/login.strategy";

  

/**

 * 动态加载所有登录策略类

 * @returns 登录策略构造函数数组

 */

export function loadLoginStrategies(): Array<new () => LoginStrategy> {

    // 使用glob查找所有策略文件

    const strategyFiles = glob.sync(join(__dirname, "../login/strategies/*.strategy.{ts,js}"));

    const strategies: Array<new () => LoginStrategy> = [];

    strategyFiles.forEach(file => {

        try {

            const exports = require(file);

            Object.values(exports).forEach(exported => {

                // 检查是否为继承自LoginStrategy的类

                if (

                    typeof exported === 'function' &&

                    exported.prototype instanceof LoginStrategy &&

                    exported !== LoginStrategy

                ) {

                    strategies.push(exported as new () => LoginStrategy);

                }

            });

        } catch (error) {

            console.error(`加载策略文件失败: ${file}`, error);

        }

    });

    return strategies;

}

  

/**

 * 创建所有策略实例

 * @returns 登录策略实例数组

 */

export function createLoginStrategyInstances(): LoginStrategy[] {

    const strategyClasses = loadLoginStrategies();

    return strategyClasses.map(StrategyClass => new StrategyClass());

}

目录:
![](./assets/loadLoginStrategies/file-20251225203356113.png)