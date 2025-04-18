# 脑电数据分析

因为脑电分析涉及多种分析方法，以及每一种方法的不同需求，本代码无法全部涉及，仅包含 ERP、频谱、时频分析的基础处理，更进一步的分析依旧可参考https://mne.tools/stable/auto_tutorials/index.html

使用jupyter notebook熟悉流程，事实上数据分析无需像预处理一样多次查看，因此制定需要的分析流程后将其转换为.py文件，并注释所有的图片显示，可批量运行

## 数据导入

使用EEG-process对脑电数据进行预处理，输出分段文件epochs

```python
# 导入预处理后的epochs数据
input_data_path = r"D:\Documents\EEG\EEG-analysis\data\preocessed_epo.fif"
epochs = mne.read_epochs(input_data_path, preload=True)
# epochs.plot(show=True)  # 查看脑电数据（可注释）
```

## 查看事件

```python
# 查看event的marker
event_id = epochs.event_id
print(event_id)
```

格式为  事件名：事件编号 

例如：{'T100': 1, 'T101': 2, 'T102': 3, 'T103': 4, 'T104': 5, 'T105': 6, 'T106': 7, 'T107': 8, 'T108': 9, 'T109': 10, 'T110': 11, 'T111': 12, 'T112': 13, 'T120': 14, 'T121': 15, 'T122': 16, 'T48': 17, 'T49': 18, 'T97': 19, 'T98': 20, 'T99': 21}

## ERP分析

```python
# ERP 
# 选择要进行分析的事件
# 如果为单个事件分析
T106 = epochs['T106'].average()  # 计算得到T101分段的均值
# 如果需要合并多个事件统一分析，首先需进行事件合并
epochs = mne.epochs.combine_event_ids(epochs, ['T100', 'T101', 'T102'], {'M': 31})  #合并'T100', 'T101', 'T102'，并将这三个事件合并后的事件命名为M，编号为31（注意，这里的编号不要跟上面输出的事件编号重叠，因此尽可能往大写）
M = epochs['M'].average()
# 绘制ERP图（全通道）
fig = T106.plot(show=False)   # 63通道的erp放在一张图里，show=False则不会有弹窗弹出erp图
fig.savefig(r"D:\Documents\EEG\EEG-analysis\output\erp.png")  # 保存fig图，按需修改路径
# 绘制ERP图（单个通道）
plt.ioff()  # 关闭交互模式
channel = epochs.info['ch_names']
for i in range(len(channel)):
    fig_channel = M.plot(picks=channel[i], show=False)
    plt.title('ERP_%s'% channel[i])
    fig_channel.savefig(r"D:\Documents\EEG\EEG-analysis\output\erp_%s.png" % channel[i])
```

如果想导出ERP数据

```python
# ERP数据输出
df = M.to_data_frame(time_format='ms')  # 输出数据时间以ms形式（可修改，time_format=None则以s形式）
df.to_excel(r"D:\Documents\EEG\EEG-analysis\output\erp.xlsx", index=False)
```

此处导出、绘制了M事件的ERP，根据所需修改事件、路径

## 频谱分析

```python
# 频谱分析
# 对于epochs数据，计算频谱（如果不对数据进行分段，分析的是静息态的整段数据也可使用，读入整段数据后，使用.compute_psd()）
evk_spectrum = M.compute_psd(method="multitaper", fmin=0, fmax=45)  # 可选welch和multitaper，tmin=10, tmax=20, fmin=5, fmax=30 时间和频率范围均可设置
fig = evk_spectrum.plot(show=False, dB=False)    # 输出频谱图
fig.savefig(r"D:\Documents\EEG\EEG-analysis\output\evk_spectrum.png")  #保存频谱图
fig2 = evk_spectrum.plot_topomap(show=False, dB=True)  # 输出频谱地形图  # dB=True 以dB的形式呈现
fig2.savefig(r"D:\Documents\EEG\EEG-analysis\output\evk_spectrum_topo.png")  #保存频谱地形图
```

此处计算的是事件M的频谱，根据需求进行修改

频谱图和频谱地形图都可以选择是否以dB为单位输出结果，compute_psd内可设置计算的时间、频率范围

若需要提取每个频段的功率谱值

```python
# 频谱 频段数据提取
psd_data = evk_spectrum.get_data()
freqs = evk_spectrum.freqs
channel = evk_spectrum.ch_names
bands = {'Delta (0-4 Hz)': (0, 4), 'Theta (4-8 Hz)': (4, 8),
         'Alpha (8-12 Hz)': (8, 12), 'Beta (12-30 Hz)': (12, 30),
         'Gamma (30-45 Hz)': (30, 45)}
psd = {}
for band_name, (fmin, fmax) in bands.items():
    freq_mask = (freqs >= fmin) & (freqs <= fmax)
    band_data = psd_data[:, freq_mask]
    band_power = band_data.mean(axis=1)
    psd[band_name] = band_power

df = pd.DataFrame(
    data=psd,          # 字典的键作为列名，值作为列数据
    index=channel      # 通道名称作为行索引
)

df = df.reset_index().rename(columns={'index': 'Channel'})
df.to_excel(r"D:\Documents\EEG\EEG-analysis\output\band_power_values.xlsx", index=False)
```

若直接提取所有频率点的功率谱值

```python
# 频谱数据输出
df = evk_spectrum.to_data_frame()
df.to_excel(r"D:\Documents\EEG\EEG-analysis\output\evk_spectrum.xlsx", index=False)  # 修改路径
```

## 时频分析

此处计算事件’T108‘的时频结果，输出保存时频图

```python
# 时频分析（morlet小波变换）
freqs = np.arange(4, 91, 1)  # 时频分析的频率范围设置：4-91Hz(不包含91)，以1Hz为步长
n_cycles = freqs / 2.0  # different number of cycle per frequency
power = epochs['T108'].compute_tfr(
    method="morlet",
    freqs=freqs,
    n_cycles=n_cycles,
    average=True,
    decim=3,
)
# average=True 即先时频之后进行平均，如果改为False则输出每一个分段的时频结果
# 输出时频图以单个通道形式输出
plt.ioff()  # 关闭交互模式
for i in range(len(power.ch_names)):
    fig_power = power.copy().plot(picks=[i], baseline=(-0.5, 0), tmin=-0.5, tmax=1, fmin=10, fmax=50, mode="logratio", title=power.ch_names[i], show=False)  # 输出时频图，以单个通道为一张图输出, 可以设置tmin=None, tmax=None, fmin=0.0, fmax=inf，设置基线，与基线的计算模式为‘mean’ | ‘ratio’ | ‘logratio’ | ‘percent’ | ‘zscore’ | ‘zlogratio’
    fig_power[0].savefig(r"D:\Documents\EEG\EEG-analysis\output\power_%s.png" % power.ch_names[i])
```

时频图绘制时，基线校准的几种模式如下：

- **mode** ‘mean’ | ‘ratio’ | ‘logratio’ | ‘percent’ | ‘zscore’ | ‘zlogratio’

  Perform baseline correction by

  subtracting the mean of baseline values (‘mean’) (default)

  dividing by the mean of baseline values (‘ratio’)

  dividing by the mean of baseline values and taking the log (‘logratio’)

  subtracting the mean of baseline values followed by dividing by the mean of baseline values (‘percent’)

  subtracting the mean of baseline values and dividing by the standard deviation of baseline values (‘zscore’)

  dividing by the mean of baseline values, taking the log, and dividing by the standard deviation of log baseline values (‘zlogratio’)

```python
# 输出power
# 基线矫正 上一部分的基线矫正是在输出时频图的过程中进行的
power_based = power.copy().apply_baseline((-0.5, 0), mode='mean')
df = power_based.to_data_frame()
df.to_excel(r"D:\Documents\EEG\EEG-analysis\output\power.xlsx", index=False)
```

输出时频结果

一些基础的脑电数据分析，输出excel仅为代码基础薄弱提供进一步分析的帮助，实际上后续的分析均可用代码实现
