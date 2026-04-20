Pillar 2 — Operations: Geographic Contagion of Delays
================
AMS 597 Statistical Computing
2026-04-20

**Question:** Which U.S. airports act as systemic super-spreaders —
propagating delays to the next aircraft leg rather than absorbing them?

**Pipeline:** EDA → Tail-Number Metrics → K-Means (K=3) → Geographic Map
→ Network Map

------------------------------------------------------------------------

## Setup

``` r
library(arrow); library(tidyverse); library(scales)
library(factoextra); library(cluster)
library(ggrepel); library(gridExtra)
library(nycflights13); library(maps)
library(igraph); library(ggraph); library(patchwork)

theme_set(theme_bw(base_size = 12))
theme_update(panel.grid.minor = element_blank(),
             plot.title       = element_text(face = "bold", size = 13),
             plot.subtitle    = element_text(color = "grey40", size = 11))

CLUSTER_COLS <- c("Outbreak Hub" = "#d62728", "Average" = "#ff7f0e", "Resilient" = "#2ca02c")
TIME_ORDER   <- c("Morning", "Afternoon", "Evening", "Night")

CHARTS <- file.path("..", "charts", "pillar2")
dir.create(CHARTS, showWarnings = FALSE, recursive = TRUE)
save_chart <- function(name, w = 13, h = 5)
  ggsave(file.path(CHARTS, name), width = w, height = h, dpi = 150, bg = "white")

df <- read_parquet(file.path("..", "data", "cleaned_flight_data_2024.parquet")) |>
  filter(Carrier_Type != "ULCC") |>
  mutate(Time_Block   = as.character(Time_Block),
         Carrier_Type = as.character(Carrier_Type),
         FlightDate   = as.Date(FlightDate))

cat(sprintf("Total flights  : %s\n", format(nrow(df), big.mark = ",")))
```

    ## Total flights  : 3,159,061

``` r
cat(sprintf("Tail numbers   : %s\n", format(n_distinct(df$Tail_Number), big.mark = ",")))
```

    ## Tail numbers   : 5,528

``` r
cat(sprintf("Origin airports: %d\n",  n_distinct(df$Origin)))
```

    ## Origin airports: 322

------------------------------------------------------------------------

## Stage 1 — EDA: System-wide cascade

``` r
cascade <- df |>
  filter(Time_Block %in% TIME_ORDER) |>
  mutate(Time_Block = factor(Time_Block, levels = TIME_ORDER)) |>
  group_by(Time_Block) |>
  summarise(mean_delay  = mean(DepDelay, na.rm = TRUE),
            pct_delayed = mean(DepDelay > 15, na.rm = TRUE) * 100,
            .groups = "drop")

mu <- mean(df$DepDelay, na.rm = TRUE)

p1 <- ggplot(cascade, aes(Time_Block, mean_delay, fill = Time_Block)) +
  geom_col(alpha = 0.85, width = 0.6, show.legend = FALSE) +
  geom_text(aes(label = sprintf("%.1f min", mean_delay)), vjust = -0.4, fontface = "bold", size = 4) +
  geom_hline(yintercept = mu, color = "forestgreen", linetype = "dashed") +
  annotate("text", x = 4.45, y = mu + 0.3, label = sprintf("Mean: %.1f min", mu),
           hjust = 1, size = 3.3, color = "forestgreen") +
  scale_fill_brewer(palette = "Blues") +
  scale_y_continuous(expand = expansion(mult = c(0, 0.15))) +
  labs(title = "Avg Departure Delay by Time Block", x = NULL, y = "Avg DepDelay (min)")

p2 <- ggplot(cascade, aes(Time_Block, pct_delayed, fill = Time_Block)) +
  geom_col(alpha = 0.85, width = 0.6, show.legend = FALSE) +
  geom_text(aes(label = sprintf("%.1f%%", pct_delayed)), vjust = -0.4, fontface = "bold", size = 4) +
  scale_fill_brewer(palette = "Oranges") +
  scale_y_continuous(expand = expansion(mult = c(0, 0.15))) +
  labs(title = "% Flights Delayed (>15 min) by Time Block", x = NULL, y = "% Delayed")

grid.arrange(p1, p2, ncol = 2)
```

![](pillar2_contagion_files/figure-gfm/p2_cascade-1.png)<!-- -->

``` r
save_chart("p2_cascade.png")
```

------------------------------------------------------------------------

## Stage 2 — Tail-Number Contagion Metrics

Three metrics per airport from consecutive same-aircraft turnarounds (≤
8 hrs):

| Metric | What it measures |
|----|----|
| **Transmission Rate** | % of delayed arrivals (\>15 min) that infect the next departure |
| **Propagation Slope** | Minutes of dep-delay generated per minute of arr-delay (`lm` slope) |
| **Propagation Factor** | Mean(next_DepDelay − ArrDelay) — positive = amplifies, negative = absorbs |

``` r
fit_slope    <- function(arr, dep) {
  ok <- !is.na(arr) & !is.na(dep)
  if (sum(ok) < 10) return(NA_real_)
  coef(lm(dep[ok] ~ arr[ok]))[2]
}
hhmm_to_min  <- function(t) (t %/% 100L) * 60L + (t %% 100L)

turnarounds <- df |>
  filter(!is.na(Tail_Number), Tail_Number != "Unknown", Tail_Number != "") |>
  arrange(Tail_Number, FlightDate, CRSDepTime) |>
  group_by(Tail_Number) |>
  mutate(next_Origin     = lead(Origin),
         next_DepDelay   = lead(DepDelay),
         next_CRSDepTime = lead(CRSDepTime)) |>
  ungroup() |>
  filter(Dest == next_Origin, !is.na(ArrDelay), !is.na(next_DepDelay)) |>
  mutate(
    arr_min      = hhmm_to_min(CRSArrTime)      + ArrDelay,
    dep_next_min = hhmm_to_min(next_CRSDepTime)  + next_DepDelay,
    dep_next_min = if_else(dep_next_min < arr_min, dep_next_min + 1440L, dep_next_min),
    turnaround_min = dep_next_min - arr_min
  ) |>
  filter(turnaround_min >= 0, turnaround_min <= 480)

tx_stats <- turnarounds |>
  group_by(Origin = Dest) |>
  summarise(
    n_turnarounds      = n(),
    n_exposed          = sum(ArrDelay > 15, na.rm = TRUE),
    n_infected         = sum(ArrDelay > 15 & next_DepDelay > 15, na.rm = TRUE),
    transmission_rate  = n_infected / pmax(n_exposed, 1),
    prop_slope         = fit_slope(ArrDelay, next_DepDelay),
    propagation_factor = mean(next_DepDelay - ArrDelay, na.rm = TRUE),
    avg_turnaround     = median(turnaround_min, na.rm = TRUE),
    .groups = "drop"
  ) |>
  filter(n_exposed >= 30)

cat(sprintf("Turnaround events : %s\n", format(nrow(turnarounds), big.mark = ",")))
```

    ## Turnaround events : 2,591,228

``` r
cat(sprintf("Airports (≥30 exp): %d\n", nrow(tx_stats)))
```

    ## Airports (≥30 exp): 300

``` r
label_df <- tx_stats |>
  filter(transmission_rate  > quantile(transmission_rate,  0.88, na.rm = TRUE) |
         avg_turnaround     < quantile(avg_turnaround,     0.10, na.rm = TRUE) |
         propagation_factor > quantile(propagation_factor, 0.90, na.rm = TRUE))

ggplot(tx_stats, aes(x = avg_turnaround, y = transmission_rate, colour = propagation_factor)) +
  geom_point(aes(size = n_turnarounds), alpha = 0.75) +
  geom_smooth(method = "loess", formula = 'y ~ x', se = TRUE, 
              colour = "grey30", fill = "grey80", linewidth = 1, show.legend = FALSE) +
  geom_text_repel(data = label_df, aes(label = Origin),
                  size = 3.2, fontface = "bold", box.padding = 0.4,
                  max.overlaps = 20, show.legend = FALSE) +
  
  scale_colour_gradient2(low = "#2166ac", mid = "gold", high = "#d62728", midpoint = 0,
                         name = "Propagation factor\n(+ amplifies / − absorbs)") +
  scale_y_continuous(labels = percent_format()) +
  scale_size_continuous(name = "Turnarounds", labels = comma, range = c(1.5, 9)) +
  labs(title    = "Transmission Rate vs Turnaround Time",
       subtitle = "Colour = Propagation Factor  |  Size = turnaround volume  |  LOESS trend",
       x = "Median Turnaround Time (min)", y = "Transmission Rate")
```

![](pillar2_contagion_files/figure-gfm/p2_transmission-1.png)<!-- -->

``` r
save_chart("p2_transmission.png", w = 11, h = 5)
```

------------------------------------------------------------------------

## Stage 3 — K-Means Clustering (K = 3)

``` r
base_stats <- df |>
  group_by(Origin) |>
  summarise(n_flights = n(), .groups = "drop") |>
  filter(n_flights >= 200)

airport_feats <- tx_stats |>
  inner_join(base_stats, by = "Origin") |>
  mutate(log_volume = log(n_flights))

FEAT_COLS <- c("transmission_rate", "prop_slope", "propagation_factor",
               "avg_turnaround",    "log_volume")

feat_mat    <- airport_feats |> select(all_of(FEAT_COLS)) |> drop_na()
feat_scaled <- feat_mat |> scale()
rownames(feat_scaled) <- airport_feats$Origin[complete.cases(airport_feats[, FEAT_COLS])]

cat(sprintf("Airports in matrix: %d\n", nrow(feat_mat)))
```

    ## Airports in matrix: 295

``` r
set.seed(42)
p_e <- fviz_nbclust(feat_scaled, kmeans, method = "wss",      k.max = 8, linecolor = "#1f77b4") +
  labs(title = "Elbow — WSS", x = "K", y = "Within-cluster SS") +
  theme(plot.title = element_text(face = "bold"))

p_s <- fviz_nbclust(feat_scaled, kmeans, method = "silhouette", k.max = 8, linecolor = "#d62728") +
  labs(title = "Silhouette Width", x = "K", y = "Avg silhouette") +
  theme(plot.title = element_text(face = "bold"))

combined_plot <- p_e + p_s
combined_plot
```

![](pillar2_contagion_files/figure-gfm/p2_kselect-1.png)<!-- -->

``` r
save_chart("p2_kselect.png", w = 12, h = 5)
```

``` r
K_CHOSEN <- 3
set.seed(42)
km <- kmeans(feat_scaled, centers = K_CHOSEN, nstart = 50, iter.max = 300)

# Label clusters by transmission_rate centroid rank
tx_idx    <- which(colnames(feat_scaled) == "transmission_rate")
rank_tx   <- rank(km$centers[, tx_idx])
label_map <- setNames(case_when(rank_tx == K_CHOSEN ~ "Outbreak Hub",
                                rank_tx == 1        ~ "Resilient",
                                TRUE                ~ "Average"),
                      seq_len(K_CHOSEN))

airport_feats <- airport_feats |>
  filter(Origin %in% rownames(feat_scaled)) |>
  mutate(cluster = factor(label_map[as.character(km$cluster)], levels = names(CLUSTER_COLS)))

cat("Cluster sizes:\n"); print(table(airport_feats$cluster))
```

    ## Cluster sizes:

    ## 
    ## Outbreak Hub      Average    Resilient 
    ##          113          179            3

``` r
FEAT_LABELS <- c(transmission_rate  = "Transmission rate",
                 prop_slope         = "Propagation slope",
                 propagation_factor = "Propagation factor",
                 avg_turnaround     = "Avg turnaround (min)",
                 log_volume         = "Log volume")

profile <- airport_feats |>
  select(cluster, all_of(FEAT_COLS)) |>
  group_by(cluster) |>
  summarise(across(everything(), \(x) mean(x, na.rm = TRUE)), .groups = "drop") |>
  pivot_longer(-cluster, names_to = "feature", values_to = "raw_mean") |>
  group_by(feature) |> mutate(z = as.numeric(scale(raw_mean))) |> ungroup() |>
  mutate(feature = factor(FEAT_LABELS[feature], levels = FEAT_LABELS))

ggplot(profile, aes(feature, cluster, fill = z)) +
  geom_tile(color = "white", lwd = 0.7) +
  geom_text(aes(label = sprintf("%.2f", raw_mean)), size = 3.5, fontface = "bold",
            color = ifelse(abs(profile$z) > 1.0, "white", "grey20")) +
  scale_fill_gradient2(low = "#2166ac", mid = "white", high = "#d73027",
                       midpoint = 0, name = "z-score") +
  scale_x_discrete(guide = guide_axis(angle = 25)) +
  scale_y_discrete(limits = rev(names(CLUSTER_COLS))) +
  labs(title = "Cluster Profile Heatmap",
       subtitle = "Red = high, Blue = low  |  values = cluster means",
       x = NULL, y = NULL) +
  theme(panel.grid = element_blank())
```

![](pillar2_contagion_files/figure-gfm/p2_cluster_profile-1.png)<!-- -->

``` r
save_chart("p2_cluster_profile.png", w = 11, h = 5)
```

------------------------------------------------------------------------

## Stage 4 — Geographic Contagion Zones

``` r
coords <- nycflights13::airports |>
  select(faa, lat, lon) |>
  filter(lon >= -130, lon <= -65, lat >= 20, lat <= 55)

map_df <- airport_feats |>
  left_join(coords, by = c("Origin" = "faa")) |>
  filter(!is.na(lat), !is.na(lon))

ggplot() +
  geom_polygon(data = map_data("state"), aes(long, lat, group = group),
               fill = "grey94", color = "white", lwd = 0.3) +
  geom_point(data = map_df |> arrange(n_flights),
             aes(lon, lat, color = cluster, size = n_flights, alpha = transmission_rate)) +
  geom_text_repel(
    data = map_df |> group_by(cluster) |> slice_max(n_flights, n = 4, with_ties = FALSE) |> ungroup(),
    aes(lon, lat, label = Origin, color = cluster),
    size = 3.2, fontface = "bold", box.padding = 0.4, max.overlaps = 40, show.legend = FALSE
  ) +
  coord_fixed(1.3, xlim = c(-125, -65), ylim = c(24, 50)) +
  scale_color_manual(values = CLUSTER_COLS, name = "Cluster") +
  scale_size_continuous(name = "Total flights", labels = comma, range = c(2, 12)) +
  scale_alpha_continuous(name = "Transmission rate", labels = percent_format(), range = c(0.35, 1)) +
  labs(title    = "U.S. Geographic Contagion Zones",
       subtitle = "Red = Outbreak Hub  |  Orange = Average  |  Green = Resilient  |  Size = flight volume",
       x = NULL, y = NULL) +
  theme(axis.text = element_blank(), axis.ticks = element_blank(),
        panel.border = element_blank(), panel.grid = element_blank())
```

![](pillar2_contagion_files/figure-gfm/p2_geo_map-1.png)<!-- -->

``` r
save_chart("p2_geo_map.png", w = 14, h = 8)
```

------------------------------------------------------------------------

## Stage 5 — Network Contagion Map

``` r
clust_lookup  <- airport_feats |> select(Origin, cluster)

cluster_edges <- df |>
  inner_join(clust_lookup, by = "Origin") |>
  inner_join(clust_lookup |> rename(Dest = Origin, dest_cluster = cluster), by = "Dest") |>
  filter(!is.na(cluster), !is.na(dest_cluster),
         as.character(cluster) != as.character(dest_cluster)) |>
  group_by(from = as.character(cluster), to = as.character(dest_cluster)) |>
  summarise(n_flights = n(), avg_dep_delay = mean(DepDelay, na.rm = TRUE), .groups = "drop")

cluster_nodes <- airport_feats |>
  group_by(name = as.character(cluster)) |>
  summarise(n_airports  = n(),
            avg_tx_rate = mean(transmission_rate,  na.rm = TRUE),
            avg_pf      = mean(propagation_factor, na.rm = TRUE),
            .groups = "drop")

g <- graph_from_data_frame(cluster_edges, vertices = cluster_nodes, directed = TRUE)

V(g)$label <- paste0(V(g)$name,
                     "\n", V(g)$n_airports, " airports",
                     "\nTx: ", percent(V(g)$avg_tx_rate, 0.1),
                     "  PF: ", round(V(g)$avg_pf, 1), " min")

ggraph(g, layout = "circle") +
  geom_edge_arc(aes(width = n_flights, color = avg_dep_delay),
                strength = 0.25, alpha = 0.85,
                arrow    = arrow(length = unit(4, "mm"), type = "closed"),
                end_cap  = circle(14, "mm")) +
  geom_node_point(aes(size = avg_tx_rate),
                  color = CLUSTER_COLS[match(V(g)$name, names(CLUSTER_COLS))], alpha = 0.9) +
  geom_node_label(aes(label = label), size = 3.5, fontface = "bold",
                  fill = "white", color = CLUSTER_COLS[match(V(g)$name, names(CLUSTER_COLS))],
                  label.size = 0.4, repel = TRUE) +
  scale_edge_color_gradient(low = "gold", high = "#d62728", name = "Avg DepDelay (min)") +
  scale_edge_width_continuous(name = "Flight volume", labels = comma, range = c(0.8, 4)) +
  scale_size_continuous(name = "Avg transmission rate", labels = percent_format(), range = c(8, 20)) +
  labs(title    = "Delay Contagion Network — Inter-Cluster Flow",
       subtitle = "Arrow width = flight volume  |  Arrow colour = avg DepDelay  |  Node size = transmission rate") +
  theme_graph(base_size = 12) +
  theme(plot.title    = element_text(face = "bold", size = 13),
        plot.subtitle = element_text(color = "grey40", size = 11))
```

![](pillar2_contagion_files/figure-gfm/p2_network-1.png)<!-- -->

``` r
save_chart("p2_network.png", w = 10, h = 8)
```

------------------------------------------------------------------------

## Summary

``` r
top_tx <- airport_feats |> slice_max(transmission_rate, n = 1)

cat("PILLAR 2 — RESULTS SUMMARY\n")
```

    ## PILLAR 2 — RESULTS SUMMARY

``` r
cat(sprintf("Airports analysed : %d  (≥200 flights, ≥30 exposures)\n", nrow(airport_feats)))
```

    ## Airports analysed : 295  (≥200 flights, ≥30 exposures)

``` r
cat(sprintf("Turnaround events : %s\n", format(nrow(turnarounds), big.mark = ",")))
```

    ## Turnaround events : 2,591,228

``` r
cat(sprintf("K-Means K         : %d\n\n", K_CHOSEN))
```

    ## K-Means K         : 3

``` r
cat(sprintf("Top super-spreader: %s  (Tx=%.1f%%  Slope=%.3f  PF=%.1f min)\n\n",
            top_tx$Origin,
            top_tx$transmission_rate * 100,
            top_tx$prop_slope,
            top_tx$propagation_factor))
```

    ## Top super-spreader: LGB  (Tx=87.4%  Slope=0.822  PF=8.2 min)

``` r
airport_feats |>
  group_by(cluster) |>
  summarise(n            = n(),
            avg_tx_rate  = percent(mean(transmission_rate,  na.rm = TRUE), 0.1),
            avg_slope    = round(mean(prop_slope,           na.rm = TRUE), 3),
            avg_pf       = round(mean(propagation_factor,   na.rm = TRUE), 2),
            avg_turnover = round(mean(avg_turnaround,       na.rm = TRUE), 1),
            .groups = "drop") |>
  print()
```

    ## # A tibble: 3 × 6
    ##   cluster          n avg_tx_rate avg_slope avg_pf avg_turnover
    ##   <fct>        <int> <chr>           <dbl>  <dbl>        <dbl>
    ## 1 Outbreak Hub   113 69.3%           0.668   2.96         48.9
    ## 2 Average        179 56.6%           0.439  -0.2          56.3
    ## 3 Resilient        3 43.6%           0.384  -2.06        413
