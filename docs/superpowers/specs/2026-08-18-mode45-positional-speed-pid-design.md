# Mode4/Mode5 等效位置式速度 PID 设计

## 目标

仅将 F 车 Mode4、Mode5 的左右轮速度控制器从增量式 PID 改为位置式 PID，并保持默认参数下的闭环 PWM 行为数学等效。Mode1、Mode2、Mode8、航向角环、角速度环和动态负压前馈不改变。

## 当前控制器

当前增量式速度 PID 的校正量为：

\[
\Delta C_k = K_p(e_k-e_{k-1}) + \Delta I_k
             + K_d(e_k-2e_{k-1}+e_{k-2})
\]

\[
\Delta I_k = \operatorname{clamp}(K_i e_k,
                                  -I_{step}, I_{step})
\]

\[
C_k = C_{k-1} +
      \operatorname{clamp}(\Delta C_k,-C_{step},C_{step})
\]

最终 PID 输出为动态前馈与闭环校正量之和：

\[
U_k = \operatorname{clamp}(FF_k+C_k,U_{min},U_{max})
\]

大角度刹车前馈 `brake_ff` 在 PID 返回之后相加并再次进行电机 PWM 限幅，本次保持该顺序不变。

## 等效位置式公式

位置式 PID 先计算绝对 P、I、D 项：

\[
e_k = r_k-y_k
\]

\[
P_k=K_p e_k
\]

\[
I_k^*=I_{k-1}+\Delta I_k
\]

\[
D_k=K_d(e_k-e_{k-1})
\]

\[
C_k^*=P_k+I_k^*+D_k
\]

在上周期执行饱和跟踪回算、满足

\[
C_{k-1}=P_{k-1}+I_{k-1}+D_{k-1}
\]

时，有：

\[
C_k^*-C_{k-1}
=K_p(e_k-e_{k-1})+\Delta I_k
+K_d(e_k-2e_{k-1}+e_{k-2})
\]

这与原增量式 PID 的 `Delta C` 完全相同。因此继续对该差值应用原有 `C_step` 限幅，就能保留原控制器触发步进限制时的 PWM 行为。

应用步进限幅和最终输出饱和后，将实际闭环校正量回算到积分状态：

\[
I_k=(U_k-FF_k)-P_k-D_k
\]

该回算维持下一周期的等效条件，并防止输出饱和后积分状态与实际 PWM 脱离。首次更新按 `e_{k-1}=0` 计算 `D_k`，以匹配原增量式 PID 的首次微分输出。

## 参数映射

完整增量式 PID 累加后的等效位置式参数映射为：

\[
K_{p,pos}=K_{p,inc},\qquad
K_{i,pos}=K_{i,inc},\qquad
K_{d,pos}=K_{d,inc}
\]

因此 Mode4、Mode5 左右轮默认值保持：

| 参数 | 默认值 |
| --- | ---: |
| Kp | 10.80 |
| Ki | 0.52 |
| Kd | 1.00 |
| I Step | 2000 PWM/周期 |
| PWM Step | 6000 PWM/周期 |

不采用 `Kp_inc -> Kd_pos` 的映射，因为它不是完整控制器的数学等效变换。

## 代码范围

1. 在 PID 核心增加一个等效位置式更新接口，保留现有 `PID_Update()` 和 `PID_UpdateIncremental()` 的行为。
2. 只有 Mode4、Mode5 左右轮速度环调用新接口。
3. 动态负压前馈继续通过 `PID_SetExternalFeedforward()` 输入；其方向、查表和限幅不变。
4. Mode4、Mode5 的 `I Limit` 菜单标签改为 `I Step`，`Inc Limit` 改为 `PWM Step`，参数注册顺序和索引保持不变。
5. 为避免含义错误，Mode4、Mode5 对应变量改名为 `speed_i_step_limit` 和 `speed_pwm_step_limit`；不增加新的可调参数。

## 饱和处理

当输出越界且本周期积分增量继续推动越界时，撤销本周期积分增量后重新计算位置式输出，保持与当前增量式控制器一致。随后对实际饱和输出执行积分回算。动态前馈不进入积分累计，前馈变化不会被错误保留为闭环积分。

## 验证标准

1. 对相同的目标速度、反馈速度和动态前馈序列运行新旧公式，逐周期比较最终 PWM；允许的差异仅为浮点舍入误差。
2. 覆盖首次启动、恒速、目标归零、正反向切换、动态前馈变化、PWM 步进限幅和电机输出饱和。
3. 检查 Mode4、Mode5 菜单索引未错位，菜单不再显示 `I Limit` 和 `Inc Limit`。
4. 执行 `git diff --check`。
5. 编译 F 车 IAR CM7_0 与 CM7_1 Debug 工程。

