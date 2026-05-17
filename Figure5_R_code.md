# R code for Figure 5

This file documents the full R code used to generate the revised conceptual Figure 5.

```r
# =========================================================
# Figure 5: Conceptual exposure-response framework
# Mean curve remains conceptual; variability ribbon is data-informed
# =========================================================

suppressPackageStartupMessages({
  library(ggplot2)
  library(dplyr)
  library(readr)
})

# --------------------------- STYLE ---------------------------
font_base       <- 12
font_axis_text  <- 9
font_axis_title <- 12
tick_len_mm     <- 1.2
axis_line_width <- 0.6
guide_lwd       <- 0.4
main_line_lwd   <- 0.9
# -------------------------------------------------------------

# ---------------------------
# 1) Read observed respiration summary
# ---------------------------
resp_sum <- read_csv("Fig2A_summary_D28_by_profile.csv",
                     show_col_types = FALSE) %>%
  mutate(
    cv = sd / mean,
    Profile_lab = gsub("\\r\\n", " ", Profile_lab)
  )

print(resp_sum %>% select(Profile, Profile_lab, n, mean, sd, cv))

cv_ambient <- resp_sum %>% filter(Profile == "abt")  %>% pull(cv)
cv_long    <- resp_sum %>% filter(Profile == "hws1") %>% pull(cv)

cv_inter <- resp_sum %>%
  filter(Profile %in% c("hws2", "hws3", "hws4")) %>%
  summarise(cv = mean(cv, na.rm = TRUE)) %>%
  pull(cv)

cv_low <- mean(c(cv_ambient, cv_long), na.rm = TRUE)

# ---------------------------
# 2) Conceptual x-axis
# ---------------------------
# These values preserve the visual geometry of the original figure.
# They should be interpreted as conceptual exposure-regime space,
# not as measured temperature.

xmin <- 14
xmax <- 40

x_sustained  <- 26.3
x_variable   <- 30.8
x_constraint <- 32.0
x_post       <- 35.0

x <- seq(xmin, xmax, by = 0.05)

# ---------------------------
# 3) Conceptual mean curve
# ---------------------------
mean_curve <- function(x,
                       x_peak = 27,
                       width_left = 7,
                       width_right = 3.2,
                       base = 0.40,
                       amp = 0.80) {
  ifelse(
    x <= x_peak,
    amp * exp(-0.5 * ((x - x_peak) / width_left)^2) + base,
    amp * exp(-0.5 * ((x - x_peak) / width_right)^2) + base
  )
}

curve_df <- data.frame(
  x = x,
  resp = mean_curve(x)
)

# ---------------------------
# 4) Data-informed variability ribbon
# ---------------------------
# Keep original left-side visual geometry.
# Only smooth the right-side contraction after the variability peak.

anchor_df <- data.frame(
  x = c(
    xmin,
    21.5,
    x_sustained,
    x_variable
  ),
  cv = c(
    cv_low   * 1.15,
    cv_low   * 0.45,
    cv_long  * 0.22,
    cv_inter * 1.05
  )
)

left_fun <- splinefun(anchor_df$x, anchor_df$cv, method = "monoH.FC")

# value at peak, used as the start of the right-side decline
peak_cv <- cv_inter * 1.05

# Smooth right-side decline only
right_fun <- function(x) {
  peak_cv / (1 + exp(0.85 * (x - x_constraint)))
}

# Smooth transition between left spline and right decline
smoothstep <- function(z) {
  z <- pmin(1, pmax(0, z))
  z * z * (3 - 2 * z)
}

rel_profile <- function(x) {
  blend_start <- x_variable - 0.8
  blend_end   <- x_variable + 1.2

  w <- smoothstep((x - blend_start) / (blend_end - blend_start))

  left_val  <- pmax(0, left_fun(pmin(x, x_variable)))
  right_val <- right_fun(x)

  (1 - w) * left_val + w * right_val
}

a <- 0.22
b <- 0.78
band_scale <- 1.00
eps <- 1e-6

band <- curve_df %>%
  mutate(
    rel = rel_profile(x),
    halfwidth = band_scale * rel * (a + b * pmax(resp, eps)),
    lower = pmax(0, resp - halfwidth),
    upper = resp + halfwidth
  )

# ---------------------------
# 5) Plot
# ---------------------------
line_moderate  <- 18
line_sustained <- 25.0
line_variable  <- 31.0

p <- ggplot(curve_df, aes(x, resp)) +
  geom_ribbon(
    data = band,
    aes(ymin = lower, ymax = upper),
    fill = "grey40",
    alpha = 0.22
  ) +
  geom_line(linewidth = main_line_lwd, colour = "black") +

  geom_vline(xintercept = line_moderate,
             linetype = 1, linewidth = 0.3, alpha = 0.6) +
  geom_vline(xintercept = line_sustained,
             linetype = 2, linewidth = guide_lwd) +
  geom_vline(xintercept = line_variable,
             linetype = 3, linewidth = guide_lwd) +

  annotate("text", x = 16.0, y = 0.36,
           label = "Low thermal\nchallenge",
           size = 2.7, fontface = "bold") +
  annotate("text", x = 21.5, y = 0.36,
           label = "Metabolic\nadjustment",
           size = 2.7, fontface = "bold") +
  annotate("text", x = 28.0, y = 0.36,
           label = "Amplified\nvariability",
           size = 2.7, fontface = "bold") +
  annotate("text", x = 35.0, y = 0.36,
           label = "Physiological\nfiltering",
           size = 2.7, fontface = "bold") +

  scale_x_continuous(
    limits = c(xmin, xmax),
    breaks = c(line_moderate, line_sustained, line_variable),
    labels = c(
      "Moderate conditions",
      "Sustained stress",
      "Variable stress"
    ),
    expand = expansion(mult = c(0.01, 0.01))
  ) +

  labs(
    x = "Cumulative thermal challenge and temporal variability",
    y = "Metabolic rate"
  ) +

  theme_bw(base_size = font_base) +
  theme(
    panel.grid.major = element_blank(),
    panel.grid.minor = element_blank(),

    panel.border = element_rect(
      colour = "black",
      linewidth = axis_line_width
    ),

    axis.text.x = element_text(
      size = font_axis_text,
      colour = "black",
      face = "bold"
    ),
    axis.title.x = element_text(
      size = font_axis_title,
      colour = "black",
      face = "bold",
      margin = margin(t = 6.5)
    ),

    axis.text.y = element_blank(),
    axis.ticks.y = element_blank(),
    axis.title.y = element_text(
      size = font_axis_title,
      colour = "black",
      face = "bold",
      margin = margin(r = 6.5)
    ),

    axis.ticks.x = element_line(linewidth = 0.4, colour = "black"),
    axis.ticks.length.x = grid::unit(tick_len_mm, "mm"),

    plot.margin = margin(2, 2, 2, 2, "mm")
  )

print(p)

# ---------------------------
# 6) Save
# ---------------------------
ggsave(
  "Figure5_conceptual_profile_variability.pdf",
  p,
  width = 135,
  height = 90,
  units = "mm",
  device = cairo_pdf
)

ggsave(
  "Figure5_conceptual_profile_variability.png",
  p,
  width = 135,
  height = 90,
  units = "mm",
  dpi = 600
)

```
