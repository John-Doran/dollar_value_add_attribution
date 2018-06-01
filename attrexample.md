# performance attribution

This is a hypothetical performance attribution on a dollar value add basis.

Assumptions:
  * four assets  (A, B, C, D)  
  * data include asset value, saa weight,  asset return, benchmark return for regular (evenly spaced) time series 
  
Let's set up the data.
  

```r
n=100 #number of periods
muddydata=FALSE
# Asset A
t=1:n
b=rnorm(n,.01,.005)  #1% perperiod expected return,.5% volatility
r=b+rnorm(n,.001,.002) #10bp tracking error, with 20bp volatility of tracking error
v1=cumprod(1+r)
v0=c(1,v1[-length(v1)])
tw=rep(.22,n)
df=data.frame(Asset="A",Time=t,Benchmark=b,Return=r,EndValue=v1,BegValue=v0,SAA_Weight=tw)
#Asset B
b=rnorm(n,.05,.1) #5% per period expected return, 10% volatility
r=b+rnorm(n,.01,.02)
v1=cumprod(1+r)
v0=c(1,v1[-length(v1)])
tw=rep(.20,n)
df=rbind(df,data.frame(
  Asset="B",Time=t,Benchmark=b,Return=r,EndValue=v1,BegValue=v0,SAA_Weight=tw))
#Asset C
b=rnorm(n,.1,.1) #10% per period expected return, 10% volatility
r=b+rnorm(n,.01,.05)
v1=cumprod(1+r)
v0=c(1,v1[-length(v1)])
tw=rep(.28,n)
df=rbind(df,data.frame(
  Asset="C",Time=t,Benchmark=b,Return=r,EndValue=v1,BegValue=v0,SAA_Weight=tw))
#Asset D
b=rnorm(n,.15,.1) #15% per period expected return, 10% volatility
r=b+rnorm(n,.01,.02)
v1=cumprod(1+r)
v0=c(1,v1[-length(v1)])
tw=rep(.3,n)
df=rbind(df,data.frame(
  Asset="D",Time=t,Benchmark=b,Return=r,EndValue=v1,BegValue=v0,SAA_Weight=tw))
```
  
calculate total fund value and returns


```r
tf=aggregate(df$EndValue,by=list(df$Time),FUN=sum)
colnames(tf)=c("Time","TFEndValue")
tfb=aggregate(df$BegValue,by=list(df$Time),FUN=sum)
colnames(tfb)=c("Time","TFBegValue")
if(muddydata) tfb$TFBegValue=tfb$TFBegValue*rnorm(n,1,.001)
tf=merge(tf,tfb)
df=merge(df,tf)
tf$TFReturn=tf$TFEndValue/tf$TFBegValue - 1
```
  
calculate total fund benchmark return and add to tf and df


```r
df$mr=df$SAA_Weight*df$Benchmark
tfb=aggregate(df$mr,by=list(df$Time),FUN=sum)
colnames(tfb)=c("Time","TFBenchmark")
tf=merge(tf,tfb)
df=merge(df,tfb)
```

Calculate total fund dollar value add for each period


```r
tf$TFDva=(tf$TFReturn-tf$TFBenchmark)*tf$TFBegValue
```

Calculate selection effect for each asset


```r
df$Selection=(df$Return-df$Benchmark)*df$BegValue
```

Add selection effects to total fund


```r
tf1=aggregate(df$Selection,by=list(df$Time),FUN=sum)
colnames(tf1)=c("Time","Selection")
tf=merge(tf,tf1)
```

Calculate allocation effects for each asset


```r
#active weight in asssets
df$Active=df$BegValue-(df$TFBegValue*df$SAA_Weight)
df$Allocation=df$Active*(df$Benchmark-df$TFBenchmark)
```

Add Allocation to total fund


```r
tf1=aggregate(df$Allocation,by=list(df$Time),FUN=sum)
colnames(tf1)=c("Time","Allocation")
tf=merge(tf,tf1)
tf$Unknown=tf$TFDva-tf$Allocation-tf$Selection
```

convert dva's to current dollars 
all effects are reinvested using total fund benchmark


```r
#calculate for whole period, but note this method works for any time subset
begmonth=1
endmonth=n
tf1=subset(tf,tf$Time>=begmonth&tf$Time<=endmonth)
df1=subset(df,df$Time>=begmonth&tf$Time<=endmonth)
trind=cumprod(1+tf1$TFBenchmark)
trmult=trind[length(trind)]/trind
trmultdf=data.frame(Time=tf1$Time,trmult)
tf1=merge(tf1,trmultdf)
df1=merge(df1,trmultdf)
df1$SelectionCD=df1$Selection*df1$trmult
df1$AllocationCD=df1$Allocation*df1$trmult
tf1$SelectionCD=tf1$Selection*tf1$trmult
tf1$AllocationCD=tf1$Allocation*tf1$trmult
tf1$TFDvaCD=tf1$TFDva*tf1$trmult
tf1$UnknownCD=tf1$Unknown*tf1$trmult
```


Now let's test that the data tie out


```r
round(sum(tf1$Allocation)-sum(df1$Allocation),4)
```

```
## [1] 0
```

```r
round(sum(tf1$AllocationCD)-sum(df1$AllocationCD),4)
```

```
## [1] 0
```

```r
round(sum(tf1$Selection)-sum(df1$Selection),4)
```

```
## [1] 0
```

```r
round(sum(tf1$SelectionCD)-sum(df1$SelectionCD),4)
```

```
## [1] 0
```

```r
round(sum(tf1$TFDva-tf1$Selection-tf1$Allocation),4)
```

```
## [1] 0
```

```r
round(sum(tf1$Unknown),4)
```

```
## [1] 0
```

```r
round(sum(tf1$TFDvaCD-tf1$SelectionCD-tf1$AllocationCD),4)
```

```
## [1] 0
```

```r
round(sum(tf1$UnknownCD),4)
```

```
## [1] 0
```
