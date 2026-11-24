# PFR-TCN-AE

> **Review-stage release:** this repository currently provides the complete pipeline pseudocode. The full source code will be released upon acceptance of the manuscript.

PFR-TCN-AE is a SCADA-based early fault detection framework for wind turbines. It combines a heteroscedastic Student-t sparse variational Gaussian process (SVGP), a power-feasible-region-constrained temporal convolutional autoencoder, causal Mahalanobis-distance monitoring, an anchored kernel density estimation (KDE) threshold, two-stage persistence confirmation, and sensor-level contribution analysis.

## Repository status

- **Available now:** review-stage pseudocode for offline training, causal online monitoring, fault localization, and event-level evaluation.
- **To be released upon acceptance:** implementation code, configuration files, environment specification, and reproduction instructions.

## Complete pipeline pseudocode

The equation numbers below refer to the revised manuscript.

```text
Algorithm 1: Offline training and online early fault detection with PFR-TCN-AE

Input:
  Raw SCADA data X; fault log F;
  W = 120 samples (120 min); alpha = 0.05; N_min = 30;
  gamma_0.93 = Q_0.93(rho_cal); epsilon = 1e-9;
  K_g = 60 samples (60 min); K_u = 120 samples (120 min);
  notification cooldown = 24 h.

Output:
  Confirmed notifications, sensor-level evidence, and detection metrics.

OFFLINE TRAINING AND CALIBRATION
1. Construct healthy windows:
   - require ControllerState = 1 and wind speed above the cut-in speed;
   - exclude shutdown, maintenance, curtailment, weather-related disturbances,
     and the periods [fault_start - 7 d, fault_end + 3 d];
   - causally impute gaps of at most 10 samples (forward fill);
   - discard windows with remaining missing values or time gaps exceeding
     1.5 sampling intervals.
2. Select 23 reconstruction targets and 18 auxiliary inputs by mutual information
   (threshold 0.2), giving 41 model inputs.
3. Define the detection periods chronologically: training (1 Jan - 10 Jun 2024),
   healthy calibration (July 2024), and online evaluation (1 Aug - 31 Dec 2024).
4. Within the training period, split chronologically into 70% SVGP training,
   10% conformal calibration, and 20% test.
5. Fit the two Student-t SVGPs (mean and log-variance) by maximizing the evidence
   lower bound; perform conformal calibration on the 10% split and construct the
   power feasible region F(x*) in Eq. (4).
6. For seeds 42, 52, 62, 72, and 82, train PFR-TCN-AE with
       L = L_data + lambda_e * L_pfr,
   where L_pfr is the SoftPlus barrier (tau = 10, m = 0.05) and lambda_e is the
   linear warm-up weight (lambda_max = 1e-3, E_w = 20 epochs).
7. From the independent healthy calibration set, estimate the residual mean mu,
   the Ledoit-Wolf covariance Sigma, the initial threshold
   T_0 = F_cal^(-1)(1 - alpha), the peak gate G_0.95 = Q_0.95(M_cal), and the
   entry coefficient gamma_0.93 = Q_0.93(rho_cal), with
   rho_t = M_t / max(tau_t, epsilon).

ONLINE CAUSAL MONITORING
8. Initialize z_t = 0, L_t = 0, M_pk = 0, and t_last = -infinity.
9. For each monitoring time t:
   a. Build the causal input window x[t-255 : t+1) (256 samples ending at t) and
      standardize it using the saved training parameters.
   b. Reconstruct the targets, obtain Yhat_t, and compute e_t = Y_t - Yhat_t.
   c. Calculate the Mahalanobis distance
        M_t = sqrt((e_t - mu)^T * Sigma^(-1) * (e_t - mu)).
   d. Form the strictly causal reference set (denoted M_t in the text)
        R_t = {M_i : t-W <= i < t and M_i < T_0}.
   e. If |R_t| < N_min, set tau_t = T_0; otherwise set tau_t to the
      (1-alpha) KDE quantile of R_t; update every 5 min and hold constant
      between updates.
   f. Set the candidate-entry threshold
        tau_tilde_t = gamma_0.93 * max(tau_t, epsilon).
   g. Set z_t = 1 if M_t > tau_tilde_t and 0 otherwise; if z_t = 1, set
      L_t = L_t + 1 and M_pk = max(M_pk, M_t); otherwise reset L_t = 0 and
      M_pk = 0 (also reset on SCADA data gaps).
   h. Set physical_gate_t = 1 if the trailing 120-min median of causal power
      residuals exceeds its healthy-calibration 99th-percentile bound, else 0
      (physical_gate_t = 1 for baseline models).
   i. Confirm an alarm at time t_c = t when either
        (L_t >= K_g and M_pk > G_0.95 and physical_gate_t = 1)
        or L_t >= K_u,
      using only information available up to t_c, without retrospective
      adjustment.
   j. Emit a notification only if t_c - t_last >= 24 h, and then set
      t_last = t_c.
   k. For each emitted notification, compute the variable contributions q_t^(i)
      in Eqs. (21)-(22) over the interval from the episode start to t_c, rank
      the mean contribution shares, and report the ten highest-ranked channels.

EVENT-LEVEL EVALUATION
10. Match notifications to the fault log within the 7-day warning window
    preceding each logged fault start.
11. Classify each event as Early, Late, or Missed; count N_E, N_L, N_M, and N_F;
    and report the event-level and point-level metrics defined in Eqs. (26)-(31)
    (N_U, FP30d, bounded point-level FPR via a_t, T_N, and Omega, median lead
    time, and fault-period AUC).
```

## Planned source-code release

Upon acceptance of the manuscript, this repository will be updated with:

- preprocessing and feature-selection scripts;
- Student-t SVGP feasible-region construction and calibration;
- PFR-TCN-AE training and inference code;
- causal KDE thresholding and two-stage persistence monitoring;
- sensor-level contribution analysis and event-matching evaluation;
- configuration files, environment specifications, and reproduction instructions.

## Data availability

The SCADA and maintenance-log data cannot be made publicly available because of confidentiality restrictions imposed by the wind-farm operator. Data access may be considered by the corresponding author upon reasonable request and subject to permission from the data owner.

## Citation

Jin et al., Early fault detection in key wind turbine components under varying operating conditions using power-feasible-region-constrained temporal convolutional autoencoder, 2026.

## Suggestions and Problem Feedback

jyr0826@163.com
