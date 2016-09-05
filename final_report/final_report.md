---
title: 'Application of the Merton Jump Diffusion Model in S&P500 '
author: "Rui Zhang 23614484"
output:
  pdf_document:
    fig_caption: yes
    number_sections: yes
    toc: yes
  html_document:
    fig_caption: yes
    number_sections: yes
    theme: flatly
    toc: yes
    toc_depth: 2
csl: ecology.csl

---

Licensed under the Creative Commons attribution-noncommercial license, http://creativecommons.org/licenses/by-nc/3.0/.
Please share and remix noncommercially, mentioning its origin.  
![CC-BY_NC](https://raw.githubusercontent.com/ruizhang-ray/531/master/cc-by-nc.png)






---
<big><big><big>Objective<big><big><big>

* Modify the Black-Scholes model, Merton jump diffusion model into the framework of POMP.

* Propose a hierarchical Merton jump diffusion model.

* Use these models fit the S&P500 data.

<br>

---

# Introduction of Data
The S&P500 data I analyzed is the close value of S&P500 from 2006-2016. The data is downloaded from Yahoo.

First we load and take a look at the data.


```r
S=read.table("SP500table.csv",header=TRUE,sep=',')
Date=S$Date
S=rev(S$Close)
plot(S,type="l")
```

<img src="figure/load_data-1.png" title="plot of chunk load_data" alt="plot of chunk load_data" style="display: block; margin: auto;" />

We can see sometimes the stock price moves a little and sometimes it takes a sharp jump.

In the following sections, I will firstly introduce the three models and describe how to modify them into the framework of POMP. Then, I will use pomp to fit these model to the data. I will focus on the comparison between the original Merton jump diffusion model and the hierarchical Merton jump diffusion model I proposed.

# Model specification
In the following models, the log returns are modeled randomly. They are independent of the passed behaviors of the stock price. Modeling the stock prices as the combination of random walk and jumps makes sense under the efficient-market hypothesis [4].

## Notation
* $\{S_t;0<t<T\}$ is the process of stock price.
* $\alpha$ is the instantaneous expected return and $\sigma$ is the instantaneous volatility.
* ${B_t}$ is a standard Brownian motion process. 

## Black-Scholes model
$$\begin{aligned}
S_t=S_0e^{(\alpha-\frac{\sigma^2}{2})t+\sigma B_t}\hspace{0.5in}[BS]
\end{aligned}
$$

Take the logarithm in both sides of the equation, we can see

$$\begin{aligned}
ln(S_t)=ln(S_0)+(\alpha-\frac{\sigma^2}{2})t+\sigma B_t.
\end{aligned}
$$
Therefore, in the Black-Scholes model, the difference of log stock price (as known as the log return) is modeled as a Brownian motion with drift.

## Merton Jump Diffusion model
$$\begin{aligned}
S_t=S_0e^{(\alpha-\frac{\sigma^2}{2}-\lambda k)t+\sigma B_t+\sum_{i=1}^{N_t}Y_i}.\hspace{0.5in}[MJD]
\end{aligned}
$$

$N_t$ is a Poisson process with rate $\lambda$, $Y_i \sim i.i.d. Normal(\mu,\delta^2)$ and $k=e^{\mu+\frac{1}{2}\sigma^2}$, $N_t$, $Y_i$ and $B_t$ are mutually independent. In *MJD*, we can see that the randomness in the log returns is modeled as a Brownian motion together with compound Poisson jump process.[1]

## Hierarchical Merton Jump Diffusion model
Compared with the *BS*, *MJD* is more volatile by adding a compound Poisson process. That is more reasonable since the stock prices do not only just walk around but also occationally move with notable jump.

One property of Poisson process is that, given the length of a time interval, the process doesn't depend on where  it starts. Mathematically, $N(t)-N(s)$ has the same distribution as $N(t-s)$. However, I think this assumption is not quite fit to the stock price. It seems to me that the prices' jumps are intense in some periods and less intense (or even disappear) in some period. Therefore, to better capture the volatitliy of the stock price, I try the following model.

Noticing that in the above two model, there is a close form of $S_t$. Therefore, we can discretize the stock price process.
$$\begin{aligned}
&I_n\sim Bernoulli(p),\\
&S_{n}=S_{n-1}e^{(\alpha-\frac{\sigma^2}{2})+\sigma B_1-I_n*(\lambda k-\sum_{i=1}^{N_1}Y_i)}.\hspace{0.5in}[HMJD]
\end{aligned}
$$

By inducing a Bernoulli random variable $I_n$, this model can seperate the stock price into intensely-jump period and no-jump period so that it can recognize the period with high volatility and the period with relative stability.

In this project, my goal is to apply POMP to answer the following two questions:

* Is the jump process a better way to model the stock price?

* Can the stock prices be recognized between the stable periods and volatile periods?

# Modify the models into POMP

Following is the POMP framework of the *HMJD*:

$$\begin{aligned}
&S_n=exp(H_n+\epsilon_n),\\
&H_n=log(S_{n-1})+(\alpha-\frac{\sigma^2}{2})+\sigma Z_n-I_n*(\lambda k-\sum_{i=1}^{N_n}Y_i),
\end{aligned}
$$
where
$$\begin{aligned}
&\{\epsilon_n\}\sim i.i.d. N(0,\eta^2),\\
&\{Z_n\} \sim i.i.d. N(0,1),\\
&I_n\sim Bernoulli(p),\\
&\{N_n\}\sim Poisson(\lambda),\\
&Y_i \sim i.i.d. N(\mu,\delta^2),\\
&k=e^{\mu+\frac{1}{2}\sigma^2}.\\
\end{aligned}
$$

* In the Black-scholes model and Merton Jump diffusion model, there are some ideal assumptions, such as the inexistence of the arbitrage oppotunity and the transaction cost, the liquidity of any share of the stocks [3]. However, this assumption doesn't always hold in the real stock market. Therefore our measurement model makes sense. ${\epsilon_n}$ is caused by the distinction between ideal and real stock market.
* Since the process of latent variable $H_n$ depends on the observation $S_n$, we need a partially plug-and-play algorithm here.[2]
* Here I only illustrate how to use POMP to describe the *HMJD*. In *BS* and *MJD*, the only difference is how to deal with the compound Poisson process term. In *BS*, it is removed and in *MJD*, it is always included.

In the following codes, I will build up several pomp object corresponding to the above 3 model.

Pomp object for *BS*:

```r
## BS

SP_statenames <- c("H","sp_state")
SP_rp_names_BS <- c("alpha","sigma","eta")
SP_ivp_names <- c("H_0")
SP_paramnames_BS <- c(SP_rp_names_BS,SP_ivp_names)
SP_covarnames <- "Co_sp"

SP_rproc1_BS <- "
  double Z;
  Z = rnorm(0,1);
  H = log(sp_state)+alpha-sigma*sigma/2+sigma*Z;
"
SP_rproc2.sim <- "
  sp_state = exp(H)*exp(rnorm(0,eta));
"

SP_rproc2.filt <- "
  sp_state = Co_sp;
"
SP_rproc_BS.sim <- paste(SP_rproc1_BS,SP_rproc2.sim)
SP_rproc_BS.filt <- paste(SP_rproc1_BS,SP_rproc2.filt)

SP_initializer <-"
  H = H_0;
  sp_state = exp(H)*exp(rnorm(0,eta));
"
SP_rmeasure <-"
  sp=sp_state;
"
SP_dmeasure <-"
  lik=dnorm(log(sp),H,eta,give_log);
"
SP_toEst_BS <-"
  Tsigma=log(sigma);
  Teta=log(eta);
"
SP_fromEst_BS <-"
  Tsigma=exp(sigma);
  Teta=exp(eta);
"

SP_BS.filt <- pomp(data=data.frame(sp=S[-1],time=1:(length(S)-1)),
                statenames=SP_statenames,
                paramnames=SP_paramnames_BS,
                covarnames=SP_covarnames,
                times="time",
                t0=0,
                covar=data.frame(Co_sp=S,time=0:(length(S)-1)),
                tcovar="time",
                rmeasure=Csnippet(SP_rmeasure),
                dmeasure=Csnippet(SP_dmeasure),
                rprocess=discrete.time.sim(step.fun=Csnippet(SP_rproc_BS.filt),
                                           delta.t=1),
                initializer=Csnippet(SP_initializer),
                toEstimationScale=Csnippet(SP_toEst_BS),
                fromEstimationScale=Csnippet(SP_fromEst_BS)
)
```

Pomp object for *MJD*:

```r
SP_rp_names_MJD <- c("alpha","sigma","lambda","mu","delta","eta")
SP_paramnames_MJD <- c(SP_rp_names_MJD,SP_ivp_names)

SP_rproc1_MJD <- "
  double N1,Z,sumY,k;
  N1 = rpois(lambda);
  Z = rnorm(0,1);
  sumY=0;
  if (N1>0){
    sumY = rnorm(N1*mu,delta*sqrt(N1));
  };
  k = exp(mu+1/2*delta*delta)-1;
  H = log(sp_state)+alpha-sigma*sigma/2-lambda*k+sigma*Z+sumY;
"

SP_rproc_MJD.sim <- paste(SP_rproc1_MJD,SP_rproc2.sim)
SP_rproc_MJD.filt <- paste(SP_rproc1_MJD,SP_rproc2.filt)

SP_toEst_MJD <-"
  Tsigma=log(sigma);
  Tdelta=log(delta);
  Tlambda=log(lambda);
  Teta=log(eta);
"
SP_fromEst_MJD <-"
  Tsigma=exp(sigma);
  Tdelta=exp(delta);
  Tlambda=exp(lambda);
  Teta=exp(eta);
"

SP_MJD.filt<- pomp(data=data.frame(sp=S[-1],time=1:(length(S)-1)),
                statenames=SP_statenames,
                paramnames=SP_paramnames_MJD,
                covarnames=SP_covarnames,
                times="time",
                t0=0,
                covar=data.frame(Co_sp=S,time=0:(length(S)-1)),
                tcovar="time",
                rmeasure=Csnippet(SP_rmeasure),
                dmeasure=Csnippet(SP_dmeasure),
                rprocess=discrete.time.sim(step.fun=Csnippet(SP_rproc_MJD.filt),
                                           delta.t=1),
                initializer=Csnippet(SP_initializer),
                toEstimationScale=Csnippet(SP_toEst_MJD),
                fromEstimationScale=Csnippet(SP_fromEst_MJD)
)
```

Pomp object for *HMJD*:

```r
SP_statenames_HMJD <- c("H","sp_state","I")
SP_rp_names_HMJD <- c("alpha","sigma","lambda","mu","delta","eta","p")
SP_ivp_names_HMJD <- c("H_0","I_0")
SP_paramnames_HMJD <- c(SP_rp_names_HMJD,SP_ivp_names_HMJD)

SP_rproc1_HMJD <- "
  double N1,Z,sumY,k;
  I = rbinom(1,p);
  N1 = rpois(lambda);
  Z = rnorm(0,1);
  sumY=0;
  if (N1>0){
    sumY = rnorm(N1*mu,delta*sqrt(N1));
  };
  k = exp(mu+1/2*delta*delta)-1;
  H = log(sp_state)+alpha-sigma*sigma/2+sigma*Z-I*(lambda*k-sumY);
"
SP_rproc_HMJD.sim <- paste(SP_rproc1_HMJD,SP_rproc2.sim)
SP_rproc_HMJD.filt <- paste(SP_rproc1_HMJD,SP_rproc2.filt)

SP_initializer_HMJD <-"
  H = H_0;
  I = 0;
  if (I_0>0.5)  I = 1;
  sp_state = exp(H)*exp(rnorm(0,eta));
"
SP_toEst_HMJD <-"
  Tsigma=log(sigma);
  Tdelta=log(delta);
  Tlambda=log(lambda);
  Teta=log(eta);
  TI_0=logit(I_0);
  Tp=logit(p);
"
SP_fromEst_HMJD <-"
  Tsigma=exp(sigma);
  Tdelta=exp(delta);
  Tlambda=exp(lambda);
  Teta=exp(eta);
  TI_0=expit(I_0);
  Tp=expit(p);
"

SP_HMJD.filt <- pomp(data=data.frame(sp=S[-1],time=1:(length(S)-1)),
                statenames=SP_statenames_HMJD,
                paramnames=SP_paramnames_HMJD,
                covarnames=SP_covarnames,
                times="time",
                t0=0,
                covar=data.frame(Co_sp=S,time=0:(length(S)-1)),
                tcovar="time",
                rmeasure=Csnippet(SP_rmeasure),
                dmeasure=Csnippet(SP_dmeasure),
                rprocess=discrete.time.sim(step.fun=Csnippet(SP_rproc_HMJD.filt),
                                           delta.t=1),
                initializer=Csnippet(SP_initializer_HMJD),
                toEstimationScale=Csnippet(SP_toEst_HMJD),
                fromEstimationScale=Csnippet(SP_fromEst_HMJD)
)
```

We can build a simulation object for the *HMJD* and take a look at the simulation results.


```r
params_test <-c(
  sigma=0.01,
  eta=0.005,
  H_0=log(1280),
  alpha=-0.0001,
  mu=0.5,
  lambda=0.002,
  delta=0.005,
  I_0=0.5,
  p=0.5
  )

sim1_HMJD.sim <-pomp(SP_HMJD.filt,
                statenames=SP_statenames_HMJD,
                paramnames=SP_paramnames_HMJD,
                covarnames=SP_covarnames,
                rprocess=discrete.time.sim(step.fun=Csnippet(SP_rproc_HMJD.sim),delta.t=1)
                )
sim <- simulate(sim1_HMJD.sim,seed=1,params=params_test,nsim=20,as=TRUE,include=TRUE)
ggplot(sim,mapping=aes(x=time,y=sp,group=sim,color=sim=='data'))+
  geom_line()+guides(color=FALSE)
```

<img src="figure/simulation_HJMD-1.png" title="plot of chunk simulation_HJMD" alt="plot of chunk simulation_HJMD" style="display: block; margin: auto;" />

We can see that the brownian motion part is quite alike the walk of the stock price. However, the compound Poisson jump process is more volatile than the jump in the stock price.

# Parameters estimation and inferences

For the following computation, we first set up different levels of computation scales. There are over 2500 observation in data. Due to the time limitation, the algorithm parameters for iterations can't be large even in level 3.

```r
run_level <-3
SP_Np = c(100,1e3,5e3)
SP_Nmif=c(10,100,200)
SP_Neval=c(10,20,30)
SP_Nglobal=c(10,10,100)
SP_Nlocal=c(10,10,20)
```

Also, let the computer prepare for the parallel computing.


```r
require(doParallel)
cores <-20
registerDoParallel(cores)
mcopts = list(set.seed=TRUE)
set.seed(931129,kind = "L'Ecuyer")
```


## Comparison between *BS* and *MJD* on the set of test parameters

I will compare the results of *BS* and *MJD* to see whether the compound Poisson jump process is preferrable to the data.

We can first see the likelihood of the test parameters.


```r
##pfilter for BS
stew(file=sprintf("pf_BS-%d.rda",run_level),{
  t_pf_BS <- system.time(
    pf_BS <- foreach(i=1:SP_Neval[run_level],.packages='pomp',
                  .options.multicore=mcopts) %dopar% try(
                    pfilter(SP_BS.filt,params=params_test[1:4],Np=SP_Np[run_level],filter.mean=TRUE)
                  )
  )
  
},seed=1320290398,kind="L'Ecuyer")

L_pf_BS <- logmeanexp(sapply(pf_BS,logLik),se=TRUE)

##pfilter for MJD
stew(file=sprintf("pf_MJD-%d.rda",run_level),{
  t_pf_MJD <- system.time(
    pf_MJD <- foreach(i=1:SP_Neval[run_level],.packages='pomp',
                  .options.multicore=mcopts) %dopar% try(
                    pfilter(SP_MJD.filt,params=params_test[1:7],Np=SP_Np[run_level],filter.mean=TRUE)
                  )
  )
  
},seed=1320290398,kind="L'Ecuyer")

L_pf_MJD <- logmeanexp(sapply(pf_MJD,logLik),se=TRUE)
```

Now, we can see the results of particle filter for *BS* and *MJD*.


```r
L_pf_BS
```

```
##                      se 
## 5646.193058    3.434801
```

```r
L_pf_MJD
```

```
##                    se 
## 5638.14952   12.39563
```

In this set of test parameters the data favors the *BS* model. However, this is just a test set, which is far from the MLEs as we can see in the simulation. Next, we will check the performance of these three model at the MLEs.

## Local search for the MLEs

Using iterated filtering [2], we can search the MLEs.

```r
SP_rw.sd_rp <- 0.02
SP_rw.sd_ivp <- 0.1
SP_cooling.fraction.50 <- 0.5

#BS
stew(file=sprintf("mif1_BS-%d.rda",run_level),{
  t.if1_BS <- system.time({
    if1_BS <- foreach(i=1:SP_Nlocal[run_level],
                   .packages='pomp', .combine=c,
                   .options.multicore=list(set.seed=TRUE)) %dopar% try(
                     mif2(SP_BS.filt,
                          start=params_test[1:4],
                          Np=SP_Np[run_level],
                          Nmif=SP_Nmif[run_level],
                          cooling.type="geometric",
                          cooling.fraction.50=SP_cooling.fraction.50,
                          transform=TRUE,
                          rw.sd = rw.sd(
                            alpha  = SP_rw.sd_rp,
                            sigma  = SP_rw.sd_rp,
                            eta    = SP_rw.sd_rp,
                            H_0    = ivp(SP_rw.sd_ivp)
                          )
                     )
                   )
    
    L.if1_BS <- foreach(i=1:SP_Nlocal[run_level],.packages='pomp',
                     .combine=rbind,.options.multicore=list(set.seed=TRUE)) %dopar% 
{
  logmeanexp(
    replicate(SP_Neval[run_level],
              logLik(pfilter(SP_BS.filt,params=coef(if1_BS[[i]]),Np=SP_Np[run_level]))
    ),
    se=TRUE)
}
  })
},seed=318817883,kind="L'Ecuyer")

r.if1_BS <- data.frame(logLik=L.if1_BS[,1],logLik_se=L.if1_BS[,2],t(sapply(if1_BS,coef)))
if (run_level>1) 
  write.table(r.if1_BS,file="sp_params_BS.csv",append=TRUE,col.names=FALSE,row.names=FALSE,sep=',')


#MJD
stew(file=sprintf("mif1_MJD-%d.rda",run_level),{
  t.if1_MJD <- system.time({
    if1_MJD <- foreach(i=1:SP_Nlocal[run_level],
                   .packages='pomp', .combine=c,
                   .options.multicore=list(set.seed=TRUE)) %dopar% try(
                     mif2(SP_MJD.filt,
                          start=params_test[1:7],
                          Np=SP_Np[run_level],
                          Nmif=SP_Nmif[run_level],
                          cooling.type="geometric",
                          cooling.fraction.50=SP_cooling.fraction.50,
                          transform=TRUE,
                          rw.sd = rw.sd(
                            alpha  = SP_rw.sd_rp,
                            mu     = SP_rw.sd_rp,
                            sigma  = SP_rw.sd_rp,
                            eta    = SP_rw.sd_rp,
                            lambda = SP_rw.sd_rp,
                            delta  = SP_rw.sd_rp,
                            H_0    = ivp(SP_rw.sd_ivp)
                          )
                     )
                   )
    
    L.if1_MJD <- foreach(i=1:SP_Nlocal[run_level],.packages='pomp',
                     .combine=rbind,.options.multicore=list(set.seed=TRUE)) %dopar% 
{
  logmeanexp(
    replicate(SP_Neval[run_level],
              logLik(pfilter(SP_MJD.filt,params=coef(if1_MJD[[i]]),Np=SP_Np[run_level]))
    ),
    se=TRUE)
}
  })
},seed=318817883,kind="L'Ecuyer")

r.if1_MJD <- data.frame(logLik=L.if1_MJD[,1],logLik_se=L.if1_MJD[,2],t(sapply(if1_MJD,coef)))
if (run_level>1) 
  write.table(r.if1_MJD,file="sp_params_MJD.csv",append=TRUE,col.names=FALSE,row.names=FALSE,sep=',')

#HMJD
stew(file=sprintf("mif1_HMJD-%d.rda",run_level),{
  t.if1_HMJD <- system.time({
    if1_HMJD <- foreach(i=1:SP_Nlocal[run_level],
                   .packages='pomp', .combine=c,
                   .options.multicore=list(set.seed=TRUE)) %dopar% try(
                     mif2(SP_HMJD.filt,
                          start=params_test,
                          Np=SP_Np[run_level],
                          Nmif=SP_Nmif[run_level],
                          cooling.type="geometric",
                          cooling.fraction.50=SP_cooling.fraction.50,
                          transform=TRUE,
                          rw.sd = rw.sd(
                            alpha  = SP_rw.sd_rp,
                            mu     = SP_rw.sd_rp,
                            sigma  = SP_rw.sd_rp,
                            eta    = SP_rw.sd_rp,
                            lambda = SP_rw.sd_rp,
                            delta  = SP_rw.sd_rp,
                            p      = SP_rw.sd_rp,
                            H_0    = ivp(SP_rw.sd_ivp),
                            I_0    = ivp(SP_rw.sd_ivp)
                          )
                     )
                   )
    
    L.if1_HMJD <- foreach(i=1:SP_Nlocal[run_level],.packages='pomp',
                     .combine=rbind,.options.multicore=list(set.seed=TRUE)) %dopar% 
{
  logmeanexp(
    replicate(SP_Neval[run_level],
              logLik(pfilter(SP_HMJD.filt,params=coef(if1_HMJD[[i]]),Np=SP_Np[run_level]))
    ),
    se=TRUE)
}
  })
},seed=318817883,kind="L'Ecuyer")

r.if1_HMJD <- data.frame(logLik=L.if1_HMJD[,1],logLik_se=L.if1_HMJD[,2],t(sapply(if1_HMJD,coef)))
if (run_level>1) 
  write.table(r.if1_HMJD,file="sp_params_HMJD.csv",append=TRUE,col.names=FALSE,row.names=FALSE,sep=',')
```

Now take a look at the results of this local search.


```r
summary(r.if1_BS$logLik,digits=5)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##  6242.4  6279.7  6301.7  6300.6  6320.9  6360.0
```

```r
summary(r.if1_MJD$logLik,digits=5)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##  6254.9  6307.3  6315.8  6319.7  6336.2  6373.9
```

```r
summary(r.if1_HMJD$logLik,digits=5)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##  6124.1  6271.1  6289.4  6305.7  6340.8  6572.1
```

In order to apply AIC, we need to compare the dimension of the parameter space of these three model.

$$\begin{aligned}
&dim(\Theta_{BS})=4\\
&dim(\Theta_{MJD})=7\\
&dim(\Theta_{HMJD})=9\\
\end{aligned}$$

Therefore, AIC can be computed using the max likelihood of the models.
$$\begin{aligned}
&AIC_{BS}=2*dim(\Theta_{BS})-2*\ell_{BS}=-12712\\
&AIC_{MJD}=2*dim(\Theta_{MJD})-2*\ell_{MJD}=-12733.8\\
&AIC_{HMJD}=2*dim(\Theta_{HMJD})-2*\ell_{HMJD}=-13126.2\\
\end{aligned}$$

We can see the *HMJD* has best performance and *BS* has the worst. 

We can plot out the variables and the likelihood to see the local geometry of the likelihood space.


```r
pairs(~logLik+log(sigma)+mu+eta+alpha+log(delta)+log(lambda)+H_0+p+I_0,data=subset(r.if1_HMJD,logLik>max(logLik)-400))
```

<img src="figure/local_geometry-1.png" title="plot of chunk local_geometry" alt="plot of chunk local_geometry" style="display: block; margin: auto;" />

##GLobal search for the MLEs

In order to investigate the whole likelihood surface and get a more convincible results, we need to start the search of MLEs from different initial value of the parameter space.

First, we set up a reasonable parameter box for randomize the initialization.


```r
SP_box <- rbind(
  sigma=c(0.005,0.1),
  alpha=c(-0.02,0.02),
  mu = c(-4,2),
  lambda=c(0.001,0.02),
  eta = c(0.01,0.1),
  delta=c(0.001,0.1),
  p=c(0.01,0.6),
  H_0 = c(6.5,8),
  I_0=c(0.2,0.8)
)
```

Again, we can apply iterated filtering for searching the MLEs. This time, the initial value of the parameters are randomly drawed from the parameter box.


```r
###BS
stew(file=sprintf("box_eval_BS-%d.rda",run_level),{
  t.box_BS <- system.time({
    if.box_BS <- foreach(i=1:SP_Nglobal[run_level],.packages='pomp',.combine=c,
                      .options.multicore=list(set.seed=TRUE)) %dopar%  
      mif2(
            SP_BS.filt,
            start=apply(SP_box[c(1,2,5,8),],1,function(x){
              ans=runif(1,x[1],x[2])
              if (min(x)>0) ans=exp(runif(1,log(x[1]),log(x[2])))
              return(ans)}),
            Np=SP_Np[run_level],
            Nmif=SP_Nmif[run_level],
            cooling.type="geometric",
            cooling.fraction.50=SP_cooling.fraction.50,
            transform=TRUE,
            rw.sd = rw.sd(
                    alpha  = SP_rw.sd_rp,
                    sigma  = SP_rw.sd_rp,
                    eta    = SP_rw.sd_rp,
                    H_0    = ivp(SP_rw.sd_ivp)
                    )
      )
    
    L.box_BS <- foreach(i=1:SP_Nglobal[run_level],.packages='pomp',.combine=rbind,
                     .options.multicore=list(set.seed=TRUE)) %dopar% {
                       set.seed(87932+i)
                       logmeanexp(
                         replicate(SP_Neval[run_level],
                                   logLik(pfilter(SP_BS.filt,params=coef(if.box_BS[[i]]),Np=SP_Np[run_level]))
                         ), 
                         se=TRUE)
                     }
  })
},seed=290860873,kind="L'Ecuyer")


r.box_BS <- data.frame(logLik=L.box_BS[,1],logLik_se=L.box_BS[,2],t(sapply(if.box_BS,coef)))
if(run_level>1) write.table(r.box_BS,file="sp500_params_box_BS.csv",append=TRUE,col.names=FALSE,row.names=FALSE)




###MJD
stew(file=sprintf("box_eval_MJD-%d.rda",run_level),{
  t.box_MJD <- system.time({
    if.box_MJD <- foreach(i=1:SP_Nglobal[run_level],.packages='pomp',.combine=c,
                      .options.multicore=list(set.seed=TRUE)) %dopar%  
      mif2(
            SP_MJD.filt,
            start=apply(SP_box[c(1:6,8),],1,function(x){
              ans=runif(1,x[1],x[2])
              if (min(x)>0) ans=exp(runif(1,log(x[1]),log(x[2])))
              return(ans)}),
            Np=SP_Np[run_level],
            Nmif=SP_Nmif[run_level],
            cooling.type="geometric",
            cooling.fraction.50=SP_cooling.fraction.50,
            transform=TRUE,
            rw.sd = rw.sd(
                            alpha  = SP_rw.sd_rp,
                            mu     = SP_rw.sd_rp,
                            sigma  = SP_rw.sd_rp,
                            eta    = SP_rw.sd_rp,
                            lambda = SP_rw.sd_rp,
                            delta  = SP_rw.sd_rp,
                            H_0    = ivp(SP_rw.sd_ivp)
                    )
      )
    
    L.box_MJD <- foreach(i=1:SP_Nglobal[run_level],.packages='pomp',.combine=rbind,
                     .options.multicore=list(set.seed=TRUE)) %dopar% {
                       set.seed(87932+i)
                       logmeanexp(
                         replicate(SP_Neval[run_level],
                                   logLik(pfilter(SP_MJD.filt,params=coef(if.box_MJD[[i]]),Np=SP_Np[run_level]))
                         ), 
                         se=TRUE)
                     }
  })
},seed=290860873,kind="L'Ecuyer")


r.box_MJD <- data.frame(logLik=L.box_MJD[,1],logLik_se=L.box_MJD[,2],t(sapply(if.box_MJD,coef)))
if(run_level>1) write.table(r.box_MJD,file="sp500_params_box_MJD.csv",append=TRUE,col.names=FALSE,row.names=FALSE)


###HMJD
stew(file=sprintf("box_eval_HMJD-%d.rda",run_level),{
  t.box_HMJD <- system.time({
    if.box_HMJD <- foreach(i=1:SP_Nglobal[run_level],.packages='pomp',.combine=c,
                      .options.multicore=list(set.seed=TRUE)) %dopar%  
      mif2(
            SP_HMJD.filt,
            start=apply(SP_box,1,function(x){
              ans=runif(1,x[1],x[2])
              if (min(x)>0) ans=exp(runif(1,log(x[1]),log(x[2])))
              return(ans)}),
            Np=SP_Np[run_level],
            Nmif=SP_Nmif[run_level],
            cooling.type="geometric",
            cooling.fraction.50=SP_cooling.fraction.50,
            transform=TRUE,
            rw.sd = rw.sd(
                            alpha  = SP_rw.sd_rp,
                            mu     = SP_rw.sd_rp,
                            sigma  = SP_rw.sd_rp,
                            eta    = SP_rw.sd_rp,
                            lambda = SP_rw.sd_rp,
                            delta  = SP_rw.sd_rp,
                            p      = SP_rw.sd_rp,
                            H_0    = ivp(SP_rw.sd_ivp),
                            I_0    = ivp(SP_rw.sd_ivp)
                    )
      )
    
    L.box_HMJD <- foreach(i=1:SP_Nglobal[run_level],.packages='pomp',.combine=rbind,
                     .options.multicore=list(set.seed=TRUE)) %dopar% {
                       set.seed(87932+i)
                       logmeanexp(
                         replicate(SP_Neval[run_level],
                                   logLik(pfilter(SP_HMJD.filt,params=coef(if.box_HMJD[[i]]),Np=SP_Np[run_level]))
                         ), 
                         se=TRUE)
                     }
  })
},seed=290860873,kind="L'Ecuyer")


r.box_HMJD <- data.frame(logLik=L.box_HMJD[,1],logLik_se=L.box_HMJD[,2],t(sapply(if.box_HMJD,coef)))
if(run_level>1) write.table(r.box_HMJD,file="sp500_params_box_HMJD.csv",append=TRUE,col.names=FALSE,row.names=FALSE)
```

It takes about 10 hours to find the global MLE for one model with run level 3.

Now, we can see the results of the global MLEs of the three models.


```r
summary(r.box_BS$logLik,digits=5)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##  6128.4  6271.8  6295.8  6292.8  6314.2  6382.3
```

```r
summary(r.box_MJD$logLik,digits=5)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##  6173.7  6272.8  6297.7  6303.6  6327.4  6569.9
```

```r
summary(r.box_HMJD$logLik,digits=5)
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##  2884.1  6265.2  6293.0  6271.4  6326.0  6684.3
```

We can compute the AIC of each model.

$$\begin{aligned}
&AIC_{BS}=2*dim(\Theta_{BS})-2*\ell_{BS}=-12756.6\\
&AIC_{MJD}=2*dim(\Theta_{MJD})-2*\ell_{MJD}=-13125.8\\
&AIC_{HMJD}=2*dim(\Theta_{HMJD})-2*\ell_{HMJD}=-13350.6\\
\end{aligned}$$

Again, the *HMJD* is chosen by the AIC. Actually, even chosen by the BIC, *HMJD* is preferred.

After investigating a larger scope of the likelihood surface, we can see the global geometry of the likelihood surface.


```r
pairs(~logLik+log(sigma)+mu+eta+alpha+log(delta)+log(lambda)+H_0+p+I_0,data=subset(r.box_HMJD,logLik>max(logLik)-400))
```

<img src="figure/global_geometry-1.png" title="plot of chunk global_geometry" alt="plot of chunk global_geometry" style="display: block; margin: auto;" />

Moreover, we can compare the diagnostics plot of the *MJD* and *HMJD*.


```r
plot(if.box_MJD)
```

<img src="figure/diagnostic-1.png" title="plot of chunk diagnostic" alt="plot of chunk diagnostic" style="display: block; margin: auto;" /><img src="figure/diagnostic-2.png" title="plot of chunk diagnostic" alt="plot of chunk diagnostic" style="display: block; margin: auto;" />

```r
plot(if.box_HMJD)
```

<img src="figure/diagnostic-3.png" title="plot of chunk diagnostic" alt="plot of chunk diagnostic" style="display: block; margin: auto;" /><img src="figure/diagnostic-4.png" title="plot of chunk diagnostic" alt="plot of chunk diagnostic" style="display: block; margin: auto;" /><img src="figure/diagnostic-5.png" title="plot of chunk diagnostic" alt="plot of chunk diagnostic" style="display: block; margin: auto;" />

* We can see this is not a very fine result. The effective sample size drops largely when there is a sudden great drop of the stock price. Some of the parameters are not convergent.

* Increasing the number of particles and the number of iteratied filitering iterations will help get a finer results. However, we should notice that to increase the effective sample size at the points of extreme drops in the stock price, we need a better model. Every model that can't model the extreme drops well can't avoid the problem of low effective sample size at the points of extreme drops of the stock price.

* We can see there are some pleasing results. In some of the iterations, the $I_n$ successfully jumps from 0 to 1 when the stock price drops sharply. This shows that, indeed, the *HMJD* can recognize the periods with volatility and the periods with relative stability. 

## More visualization of the likelihood surface.

We can treat $\alpha$ and $\mu$ as the only two parameters, by fixing the other parameters at the value of the MLEs from *HMJD*. In this way, the likelihood and the parameters together are three dimensional space, which can be shown directly.

This two parameters basically determine the trend of the stock prices. Seeing the likelihood surface of these two parameters, we can have a feel of which direction the stock price walks and jumps in. 


```r
para=r.box_HMJD[which.max(r.box_HMJD$logLik),3:11]
s=50
expand.grid(alpha=seq(from=-0.005,to=0.01,length=s),
            mu=seq(from=-0.5,to=0.5,length=s)) -> p
p_grid=cbind(p,(sapply(para[c(1,4:9)],rep,s*s)))


stew(file=sprintf("lik_grid23_HJMD_1_%d.rda",run_level),
     {p <- foreach(i=1:(s*s),.packages='pomp',.combine=rbind,
                      .options.multicore=list(set.seed=TRUE)) %dopar% try(
 {
   pfilter(SP_HMJD.filt, params=unlist(p_grid[i,]),Np=5000) -> pf
   theta=p_grid[i,]
   theta$loglik <- logLik(pf)
   theta
 })
 },seed=290860873,kind="L'Ecuyer")

require(dplyr)
pp <- mutate(p,loglik=ifelse(loglik>max(loglik)-600,loglik,NA))
ggplot(data=pp,mapping=aes(x=alpha,y=mu,z=loglik,fill=loglik))+
  geom_tile(color=NA)+
  geom_contour(color='black',binwidth=100)+
  geom_point(aes(x=0,y=0),color="red")+
  scale_fill_gradient()+
  labs(x=expression(alpha),y=expression(mu))
```

<img src="figure/likelihood_surface-1.png" title="plot of chunk likelihood_surface" alt="plot of chunk likelihood_surface" style="display: block; margin: auto;" />

In the top region of the contour, we can see, most of $\mu's$ are below zero and most of $\alpha's$ are above zero. Therefore, we can infer that averagely, stock price jumps downward and walks upward. 

We can further investigate the profile likelihood of alpha. Using iterated filtering, we can find the MLEs for each fixed $\alpha$, then we got the profile likelihood of $\alpha$.


```r
## construct_parameter_box
It=30
nprof=40
profile.box <- profileDesign(  
  alpha=seq(-0.01,0.01,length.out=It),
  lower=SP_box[-2,1],
  upper=SP_box[-2,2],
  nprof=nprof
)

## mif1
stew(file=sprintf("profile alpha-1-%d.rda",It),{
  
  t_global.prf1 <- system.time({
      prof.llh<- foreach(i=1:(It*nprof),.packages='pomp', .combine=rbind, .options.multicore=mcopts) %dopar%{
        # Find MLE
        mif2(
            SP_HMJD.filt,
            start=unlist(profile.box[i,]),
            Np=3000,
            Nmif=50,
            cooling.type="geometric",
            cooling.fraction.50=SP_cooling.fraction.50,
            transform=TRUE,
            rw.sd = rw.sd(
                            mu     = SP_rw.sd_rp,
                            sigma  = SP_rw.sd_rp,
                            eta    = SP_rw.sd_rp,
                            lambda = SP_rw.sd_rp,
                            delta  = SP_rw.sd_rp,
                            p      = SP_rw.sd_rp,
                            H_0    = ivp(SP_rw.sd_ivp),
                            I_0    = ivp(SP_rw.sd_ivp)
                    )
        )->mif_prf
        # evaluate llh
        evals = replicate(10, logLik(pfilter(mif_prf,Np=3000)))
        ll=logmeanexp(evals, se=TRUE)        
        
        data.frame(as.list(coef(mif_prf)),
                   loglik = ll[1],
                   loglik.se = ll[2])
      }
  })
},seed=931129,kind="L'Ecuyer")

t_global.prf1
```

```
##       user     system    elapsed 
## 615817.180     33.426  33239.888
```

```r
## filiter again on the maxima

prof.llh %>%   
  mutate(alpha=signif(alpha,digits=6))%>%
  ddply(~alpha,subset,rank(-loglik)<=10) %>%
  subset(select=SP_paramnames_HMJD) -> pars


## mif2 again
stew(file=sprintf("profile alpha-2-%d.rda",It),{
  
  t_global.prf2 <- system.time({
    prof.llh<- foreach(i=1:(nrow(pars)),.packages='pomp', .combine=rbind, .options.multicore=mcopts) %dopar%{
      # Find MLE
      mif2(
            SP_HMJD.filt,
            start=unlist(pars[i,]),
            Np=5000,
            Nmif=100,
            cooling.type="geometric",
            cooling.fraction.50=SP_cooling.fraction.50,
            transform=TRUE,
            rw.sd = rw.sd(
                            mu     = SP_rw.sd_rp,
                            sigma  = SP_rw.sd_rp,
                            eta    = SP_rw.sd_rp,
                            lambda = SP_rw.sd_rp,
                            delta  = SP_rw.sd_rp,
                            p      = SP_rw.sd_rp,
                            H_0    = ivp(SP_rw.sd_ivp),
                            I_0    = ivp(SP_rw.sd_ivp)
                    )
      )->mif_prf2
      # evaluate llh 
      pf= replicate(10,pfilter(mif_prf2,Np=5000))
      evals=sapply(pf,logLik)
      ll=logmeanexp(evals, se=TRUE)  
      nfail=sapply(pf,getElement,"nfail")
      
      data.frame(as.list(coef(mif_prf2)),
                 loglik = ll[1],
                 loglik.se = ll[2],
                 nfail.max=max(nfail))
    }
  })
},seed=931129,kind="L'Ecuyer")
t_global.prf2
```

```
##       user     system    elapsed 
## 473027.386    170.524  25173.847
```

```r
## plot_profile
prof.llh %<>%
  mutate(alpha=signif(alpha,digits=6))%>%
  ddply(~alpha,subset,rank(-loglik)<=2)

a=prof.llh$loglik[which(rank(-prof.llh$loglik)==5)]

prof.llh %>%
  ggplot(aes(x=alpha,y=loglik))+
  geom_point()+
  geom_smooth(method="loess")+
  geom_hline(aes(yintercept=a),linetype="dashed",color="red")
```

<img src="figure/profile_alpha-1.png" title="plot of chunk profile_alpha" alt="plot of chunk profile_alpha" style="display: block; margin: auto;" />

* We can see the $\alpha$ attains the MLE around 0.001, which also justifies the result we see from the log likelihood surface.

* The likelihood of the MLE is much higher than the likelihood of others. The reason of this might be the likelihood surface are sharp around the MLE or there is computational (numerical) error. According to the high likelihood around the MLE (the points above the red dashed line), I believe it is the former case.

* Here, I didn't draw the confidence interval since the MLE is the only one inside the confidence interval.

* Due to the computational limitation, this profile likelihood is quite rough. To see a finer profile likelihood, we need to increase the numbers of iterations. 

# Conclusion and discussion on the improvement

* First, I want to explain why I use all the observations of ten years. Actually, I tried to run the whole research on only about 150 observations and found that the likelihoods of these three models are very close. Therefore it is more reasonable for us to discuss these three models over a long time scale. 

* From the results and analysis above, we can  conclude that the data of S&P500 favors *MJD* and *HMJD*, the models with Compound Poisson jump process, more than *BS*, the models without jump process. Moreover, *HMJD* has the best AIC therefore it is the best model among the three model. From the diagnostic plot, we can see the $I_n$ in some samples indeed succcessfully recognizes the distinction between different periods of the stock price.

* However, there are quite a lot $I_n$ that fail to recognize the periods with high volatility. To improve the results, we need a computation level with larger iterations. Also we might need a smaller random walk standard deviation, since from the profile likelihood of $\alpha$, we see that the peaks in the likelihood surface might be very narrow.

* To improve the model, we can try to model $I_n$ as a random variable with categorical distribution. By doing so, we can subdivide the volatile periods of the stock price into different levels. Their distinctions can be shown by their own $\lambda$, $\mu$ and $\delta$.

* To increase the effective sample size at the time points where the stock price drops sharply, we need a better model or method to deal with this kind of rare events.


-----
# Reference

* [1] Kazuhisa Matsuda. 2004. Introduction to Merton Jump Diffusion Model.

* [2] Class notes of Stats 531 (Winter 2016) 'Analysis of Time Series', instructor: Edward L. Ionides (http://ionides.github.io/531w16/).

* [3] Geometric Brownian motion on Wikipedia (https://en.wikipedia.org/wiki/Geometric_Brownian_motion).

* [4] Efficient-market hypothesis on Wikipedia (https://en.wikipedia.org/wiki/Efficient-market_hypothesis)
