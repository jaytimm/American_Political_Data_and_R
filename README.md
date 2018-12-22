Federal election data & R: some resources & methods
---------------------------------------------------

A collection of political data resources. Many of the data collated here should be more easily & publicly accessible. It is not clear why they are not.

Lots of help from the folks at ....

-   [1 Lawmaker details](#1-Lawmaker-details)
-   [2 Political Ideologies](#2-political-ideologies-and-congressional-composition)
-   [3 Political geometries](#4-political-geometries)
-   [4 Federal election results](#5-Federal-election-results)
-   [5 Census data and congressional districts](#6-Census-data-and-congressional-districts)
-   [6 Alternative geometries](#7-Funky-geometries)
-   [7 A work in progress](#8-A-work-in-progress)

Some additional text. A bit of a layman's guide to working with federal election data using R. I am posting this on Git Hub (as opposed to my website/blog) as I hope to develop this as a resource.

``` r
library(tidyverse)
```

------------------------------------------------------------------------

### 1 Lawmaker details

> [CivilServiceUSA](https://github.com/CivilServiceUSA) provides a wonderful collection of details about each lawmaker in the 115th Congress, including age, race, religion, biographical details, and social media info. A full roll call of information available for each lawmaker is available [here](https://github.com/CivilServiceUSA/us-house#data-set). Presumably other resources exist for accessing this type of information, but this particular resource is super rich/convenient. Tables can be downloaded directly from their Git Hub site. I prefer the json format.

``` r
csusa_senate_dets <- jsonlite::fromJSON(url('https://raw.githubusercontent.com/CivilServiceUSA/us-senate/master/us-senate/data/us-senate.json'))
csusa_house_dets <- jsonlite::fromJSON(url('https://raw.githubusercontent.com/CivilServiceUSA/us-house/master/us-house/data/us-house.json'))
```

Here, we consider some different perspectives on the composition of the 115th House utilizing these data.

#### 1.1 Age & generational demographics of the 115th House

``` r
csusa_house_dets %>%
  mutate (years = 
            lubridate::year(as.Date(Sys.Date())) -
            lubridate::year(as.Date(date_of_birth))) %>%
  ggplot (aes(years)) +
  geom_histogram(bins=20, fill = 'steelblue', alpha = .85) +
  labs(title = '115th House composition by age',
       caption = 'Data source: CivilServiceUSA')
```

![](README_files/figure-markdown_github/unnamed-chunk-3-1.png)

> Pew Research has seemingly taken a leadership role in formally [delineating generations](http://www.pewresearch.org/fact-tank/2018/04/11/millennials-largest-generation-us-labor-force/ft_15-05-11_millennialsdefined/), which have always been hazy, sources of contention, and good clean American fun. ... the beauty of generation naming is that it is truly a crowd-sourced effort.

Generations in congress~

-   Millenials: 1981-1997
-   Generation X: 1965 -1980
-   Baby Boomers: 1946-1964
-   Silent: 1928-1945
-   Greatest: &lt; 1928

I take some liberties here with this classfication, as I have issues with the duration of the Boomer generation. Namely: (a) Boomers-proper 1946-1954 & (b) Generation Jones 1955-1964.

``` r
gens115 <- csusa_house_dets %>%
  mutate (yob = as.numeric(gsub('-.*$', '', date_of_birth))) %>%
  mutate (gen = case_when (yob < 1998 & yob > 1980 ~ '4- Millenial',
                           yob < 1981 & yob > 1964 ~ '3- Gen X',
                           yob < 1965 & yob > 1954 ~ '2b - Gen Jones',
                           yob < 1955 & yob > 1945 ~ '2a - Boomer-proper',
                           yob < 1946 & yob > 1927 ~ '1 - Silent'))
gens115 %>%
  group_by(gen,party) %>%
  summarize(n=n()) %>%
  group_by(party) %>%
  mutate(rank = row_number())%>%
  ggplot(aes(x=reorder(gen, -rank), 
             y=n, 
             fill=gen)) + 
  geom_col(show.legend = FALSE, alpha = 0.85)+
  ggthemes::scale_fill_stata() +
  xlab(NULL) + ylab(NULL) +
  facet_wrap(~party) +
  coord_flip() +
  labs(title = '115th US House composition by generation',
       caption = 'Data source: CivilServiceUSA')
```

![](README_files/figure-markdown_github/unnamed-chunk-4-1.png)

#### 1.2 Religion

And perhaps a look at religion for good measure.

``` r
cols <- RColorBrewer::brewer.pal(4, 'Set1')
cols <- colorRampPalette(cols)(31)

csusa_house_dets %>%
  group_by(religion) %>%
  summarize(n = n()) %>%
  na.omit() %>%
    ggplot(aes(area = n,
               fill = religion,
               label = religion,
               subgroup = religion)) +
      treemapify::geom_treemap(alpha=.85) +
      treemapify::geom_treemap_subgroup_border() +
      treemapify::geom_treemap_text(colour = "white", 
                        place = "topleft", 
                        reflow = T,
                        size = 11)+
      scale_fill_manual(values = cols) +
      theme(legend.position = "none",
            #plot.title = element_text(size=12),
            legend.title=element_blank()) +
      labs(title = '115th House composition by religion',
           caption = 'Source: CivilServiceUSA')
```

![](README_files/figure-markdown_github/unnamed-chunk-5-1.png)

So, some simple examples of what can be with .... And here's hoping they continue their project into the new (116th) Congress.

------------------------------------------------------------------------

### 2 Political ideologies and congressional composition

> The [VoteView](https://voteview.com/) project has been the standard for ... I oddly enough became hip to their methods via linguisti

``` r
#sen115 <- Rvoteview:: member_search(chamber= 'Senate', congress = 115)

rvoteview_house_50 <- lapply(c(66:115), function (x)
                    Rvoteview::member_search (
                      chamber = 'House', 
                      congress = x)) %>% 
  bind_rows()
```

#### 2.1 Congressional composition

A bit of a viz. Note that these percentages are not erfect, as non-major political parties are not included (which comprise a very small overall peracentage).

``` r
rvoteview_house_50 %>%
  filter(party_name %in% c('Democratic Party', 'Republican Party')) %>%
  group_by(congress, party_name) %>%
  summarize(n = n()) %>%
  mutate(n = n/sum(n)) %>%
  ggplot(aes(x=congress, y=n, fill = party_name)) +
  geom_area(alpha = 0.85, color = 'gray') +
  ggthemes::scale_fill_stata()+
  geom_hline(yintercept = 0.5, color = 'white', linetype = 2) +
  theme(legend.position = "bottom")+
  labs(title = "House Composition over the last 50 congresses",
       caption = 'VoteView')
```

![](README_files/figure-markdown_github/unnamed-chunk-7-1.png)

#### 2.2 Political ideologies historically

``` r
rvoteview_house_50 %>%
  filter(congress > 89) %>%
    ggplot(aes(x=nominate.dim1, y=as.factor(congress), fill = congress)) +
    ggridges::geom_density_ridges(rel_min_height = 0.01) +
    geom_vline(xintercept = 0, color = 'black', linetype = 2) +
    theme(legend.position = "none") + 
    ylab("")+
    labs(title = "Political ideologies in US Houses 90 to 115",
         caption = 'VoteView')
```

![](README_files/figure-markdown_github/unnamed-chunk-8-1.png)

#### 2.3 NOKKEN & POOLE scores

An alternative approach. --- Voteview data with

(that change per congress). Scores via `Rvoteview` only DW\_Nominate, which reflect an aggregate score based on lawmaker's entire voting history (eben if they switch houses, which is weird).

Mention the `bioguide` which helps cross.

``` r
voteview_house115 <- read.csv(url("https://voteview.com/static/data/out/members/HSall_members.csv"),
  stringsAsFactors = FALSE) %>%
  filter(chamber == 'House' & congress == 115)
```

------------------------------------------------------------------------

### 3 Political geometries

> The `tigris` package provides a super convenient interface ... for loading US shapefiles ...

``` r
nonx <- c('78', '69', '66', '72', '60', '15', '02')

library(tigris); options(tigris_use_cache = TRUE, tigris_class = "sf")
us_house_districts <- tigris::congressional_districts(cb = TRUE) %>%
  select(GEOID,STATEFP, CD115FP) %>%
  
  left_join(tigris::states(cb = TRUE) %>% 
              data.frame() %>%
              select(STATEFP, STUSPS)) 

laea <- sf::st_crs("+proj=laea +lat_0=30 +lon_0=-95") # Lambert equal area
us_house_districts <- sf::st_transform(us_house_districts, laea)
```

------------------------------------------------------------------------

### 4 Federal election results

> [Daily Kos data sets](https://www.dailykos.com/stories/2018/2/21/1742660/-The-ultimate-Daily-Kos-Elections-guide-to-all-of-our-data-sets)

Not fantastic structure-wise. Some lawmaker bio details (Name, First elected, Birth Year, Gender, RAce/ethnicity, Religion, LGBT). House sheet: 2016/2012/2008 presidential election results by congressional district; along with 2016/2014 house congressional results; No 2018 results.

Also includes some socio-dems by district, but this is likely more easily addressed using `tidycensus`.

#### 4.1 Restructuring election data

``` r
url <- 'https://docs.google.com/spreadsheets/d/1oRl7vxEJUUDWJCyrjo62cELJD2ONIVl-D9TSUKiK9jk/edit#gid=1178631925'

house <- gsheet::gsheet2tbl(url) 
```

Data are super dirty. A simple cleaning procedure that will scale (for the most part) to other data sources at the Daily Kos. With a simple focus on ... :

``` r
fix <- as.data.frame(cbind(colnames(house), as.character(house[1,])), 
  string_as_factor = FALSE) %>%
  mutate(V1 = gsub('^X', NA, V1)) %>%
  fill(V1) %>%
  mutate(nw_cols = ifelse(is.na(V2), V1, paste0(V1, '_', V2)),
         nw_cols = gsub(' ', '_', nw_cols))

colnames(house) <- fix$nw_cols
house <- house %>% slice(3:nrow(.))
keeps <- house[,!grepl('Pronun|ACS|Census|Survey', colnames(house))]
```

Here we filter to data to the last three Presidential elections.

``` r
dailykos_pres_elections <- keeps [,c('District', 'Code', grep('President_[A-z]', colnames(house), value=T))] %>%
  gather (key = election, value = percent, `2016_President_Clinton`:`2008_President_McCain`) %>%
  mutate(election = gsub('President_', '', election),
         percent = as.numeric(percent)) %>%
  separate(Code, c('STUSPS', 'CD115FP')) %>%
  separate(election, c('year', 'candidate'))%>%
  mutate(CD115FP = ifelse(CD115FP == 'AL', '00', CD115FP)) %>%
  left_join(data.frame(us_house_districts) %>% select (-geometry))
```

#### 4.2 Presidential Election results - 2016

``` r
us_house_districts %>%
  filter(!gsub('..$' ,'', GEOID) %in% nonx) %>%
  left_join(dailykos_pres_elections %>% 
              filter(year == '2016') %>%
              spread(candidate, percent) %>%
              mutate(dif = Trump-Clinton)) %>%
  ggplot() + 
  geom_sf(aes(fill = dif)) +
  scale_fill_distiller(palette = "RdBu",direction=-1)+
  theme(axis.title.x=element_blank(),
        axis.text.x=element_blank(),
        axis.title.y=element_blank(),
        axis.text.y=element_blank(),
        legend.position = 'bottom') +
  labs(title = "Trump support - Clinton support",
       subtitle = '2016 Presidential Elections',
       caption = 'Data source: Daily Kos')
```

![](README_files/figure-markdown_github/unnamed-chunk-14-1.png)

#### 4.3 Rural & urban voting

Using the area of congressional districts (in log square meters) as a proxy for degree of urbanity. ... Plot Trump support as function of CD area.

``` r
us_house_districts %>%
  left_join(dailykos_pres_elections %>%
              filter(candidate == 'Trump')) %>%
  mutate(area = as.numeric(gsub(' m^2]', '', sf::st_area(.)))) %>%
  ggplot(aes(percent, log(area))) +
  geom_point(color = 'steelblue') +
  geom_smooth(method="loess", se=T, color = 'darkgrey')+
  labs(title = "2016 Trump support vs. log(area) of congressional district",
       caption = 'Source: Daily Kos')
```

![](README_files/figure-markdown_github/unnamed-chunk-15-1.png)

------------------------------------------------------------------------

### 5 Census data and congressional districts

> Using `tidycensus` ... Investigating educational attainment by race.

#### 5.1 Acquire census data

Census race/ethnicity per US Census classifications.

``` r
code <- c('A', 'B', 'C', 'D', 'E',
          'F', 'G', 'H', 'I')
          
          
race <- c('WHITE ALONE', 'BLACK OR AFRICAN AMERICAN ALONE',
          'AMERICAN INDIAN OR ALASKAN NATIVE ALONE',
          'ASIAN ALONE', 
          'NATIVE HAWAIIAN AND OTHER PACIFIC ISLANDER ALONE', 
          'SOME OTHER RACE ALONE', 'TWO OR MORE RACES',
          'WHITE ALONE, NOT HISPANIC OR LATINO',
          'HISPANC OR LATINO')

race_table <- as.data.frame(cbind(code,race),
                            stringsAsFactors=FALSE)
```

C15002: SEX BY EDUCATIONAL ATTAINMENT FOR THE POPULATION 25 YEARS AND OVER

``` r
search_vars <- var_list[grepl('C1500', var_list$name),]

tidycens_data <- tidycensus::get_acs(geography = 'congressional district',
                            variables = search_vars$name,
                            summary_var = 'B15002_001',
                            year = 2017,
                            survey = 'acs5') %>%
  left_join(search_vars %>% rename(variable = name)) %>%
  filter(!grepl('Total$|Female$|Male$', label)) %>%
  
  mutate(gender = ifelse(grepl('Male', label), 'Male', 'Female'),
         label = gsub('^Estimate.*!!', '', label),
         code = gsub('(C[0-9]+)([A-Z])(_[0-9]+.$)', 
                     '\\2', 
                     variable)) %>%
  left_join (race_table) %>%
  select(GEOID, label, gender, race, estimate:summary_moe)
```

#### 5.2 White males without college degrees

White men without college degree. As percentage of total population over 25. ie, as a percentage of the electorate. Also -- map zoomed into some interesting sub0location. NEED to re-project.

``` r
us_house_districts %>% 
  filter(!gsub('..$' ,'', GEOID) %in% nonx) %>%
  left_join(tidycens_data %>% 
              filter(label != 'Bachelor\'s degree or higher' &
                     gender == 'Male' & 
                     race == 'WHITE ALONE, NOT HISPANIC OR LATINO')) %>%
  mutate(per = estimate / summary_est) %>%
  ggplot() + 
  geom_sf(aes(fill = per)) + 
  
  scale_fill_distiller(palette = "BrBG",direction=1)+
  
  theme(axis.title.x=element_blank(),
        axis.text.x=element_blank(),
        axis.title.y=element_blank(),
        axis.text.y=element_blank(),
        legend.position = 'bottom') +
  labs(title = "Percentage of White males w/o a college degree by congressional district",
       caption = 'Source: American Community Survey, 5-Year estimates, 2013-17, Table C15002')
```

![](README_files/figure-markdown_github/unnamed-chunk-20-1.png)

#### 5.3 Educational attainment profiles by CD

Create plots of some cherry-picked district cross-sections (per Daily Kos).

Definitions: WHITE ALONE means/equals all the whites, hispanic or otherwise. OR, WHITE ALONE, HISPANIC + WHITE ALONE, NOT HISPANIC.

``` r
tree <- tidycens_data %>%
  left_join(data.frame(us_house_districts) %>% select(GEOID, STUSPS, CD115FP)) %>%
  mutate (race = gsub(', | ', '_', race)) %>%
  select(-moe:-summary_moe) %>%
  spread(race, estimate) %>%
  mutate(WHITE_ALONE_HISPANIC = WHITE_ALONE - WHITE_ALONE_NOT_HISPANIC_OR_LATINO) %>%
  gather(key =race, value = estimate, AMERICAN_INDIAN_OR_ALASKAN_NATIVE_ALONE:WHITE_ALONE_HISPANIC) %>%
  filter(race != 'HISPANIC OR LATINO') %>%
  mutate(race_cat = ifelse(race == 'WHITE_ALONE_NOT_HISPANIC_OR_LATINO', 'White', 'Non-White'),
    ed_cat = ifelse(label == 'Bachelor\'s degree or higher', 'College', 'Non-College'))%>%
  group_by(GEOID, STUSPS, CD115FP, race_cat, ed_cat) %>%
  summarize(estimate = sum(estimate)) %>%
  group_by(GEOID) %>%
  mutate(per = estimate/sum(estimate)) %>%
  ungroup()
```

Non-College White share. WE should check how this is calculated. Or at least define how we define it.

Also: WE need to cross GEOID to actual state/congressional districts for some reference.

``` r
samp_n <- sample(unique(tree$GEOID), 12)

tree %>%
  filter(GEOID %in% samp_n) %>%
    ggplot(aes(area = per,
               fill = paste0(race_cat, ' ', ed_cat),
               label = paste0(race_cat, ' ', ed_cat),
               subgroup = paste0(race_cat, ' ', ed_cat)))+
      treemapify::geom_treemap(alpha=.85)+
      treemapify::geom_treemap_subgroup_border() +

      treemapify::geom_treemap_text(colour = "white", 
                        place = "topleft", 
                        reflow = T,
                        size = 9)+
      #ggthemes::scale_fill_stata()+ 
      scale_fill_brewer(palette = 'Paired') +
      facet_wrap(~paste0(STUSPS, '-', CD115FP)) +
      theme(legend.position = "bottom",
            legend.title=element_blank()) + 
      labs(title = "Educational attainment by race for population over 25",
           subtitle = 'A random sample of congressional districts',
           caption = 'Source: American Community Survey, 5-Year estimates, 2013-17, Table C15002')
```

![](README_files/figure-markdown_github/unnamed-chunk-22-1.png)

#### 5.4 Trump support by educational attainment

Trump ed/race dems by binned degrees of support.

``` r
ed_45 <- dailykos_pres_elections %>%
  filter(candidate == 'Trump') %>%
  group_by(candidate) %>%
  mutate(cut = cut_number(percent, n =10),
         rank_cut = dense_rank(cut)) 
```

A summary of congressional districts:

``` r
table(ed_45$cut)
```

    ## 
    ##  [4.9,21.4] (21.4,30.5] (30.5,36.6] (36.6,43.1] (43.1,48.7] (48.7,53.1] 
    ##          44          46          42          42          45          42 
    ## (53.1,56.2] (56.2,60.9] (60.9,65.6] (65.6,80.4] 
    ##          44          44          42          44

Describe plot. The average educational attainment profile for each level (ie, bin) of support for 45.

``` r
ed_45 %>%
  left_join(tree) %>%
  mutate(type = paste0(race_cat, ' ', ed_cat)) %>%
  select(type, rank_cut, cut, estimate) %>%
  group_by(type, cut, rank_cut) %>%
  summarize(estimate = sum(estimate)) %>%
  group_by(cut)%>%
  mutate(new_per = estimate/sum(estimate)) %>%
  
ggplot(aes(x=(rank_cut), y=new_per, fill = type)) +
  geom_area(alpha = 0.75, color = 'gray') +
  scale_fill_brewer(palette = 'Paired') +
  scale_x_continuous(breaks = 1:10, labels = 1:10) + 
  theme(legend.position = "bottom")+
  labs(title = "Educational attainment profiles by level of support for 45",
       subtitle = '2016 Presidential Election')+
  xlab('Level of support for 45')+ylab(NULL)
```

![](README_files/figure-markdown_github/unnamed-chunk-25-1.png)

------------------------------------------------------------------------

### 6 Alternative political geometries

> The Daily Kos makes available of set of shapefiles meant to represent congressional districts and states as .... The Daily Vos makes these shapefiles availble via Google Drive. Links are provided below.

``` r
#Hex map
dailyvos_hex_cd <- 'https://drive.google.com/uc?authuser=0&id=1E_P0r1Uv438fZsvKsvidIR02Nb5Ju9zf&export=download/HexCDv12.zip'
dailyvos_hex_st <- 'https://drive.google.com/uc?authuser=0&id=0B2X3Bx1aCHsJVWxYZGtxMGhrMEE&export=download/HexSTv11.zip'
#Tile map
dailyvos_tile_outer <- 'https://drive.google.com/uc?authuser=0&id=0B2X3Bx1aCHsJdGF4ZWRTQmVyV2s&export=download/TileOutv10.zip'
dailyvos_tile_inner <- 'https://drive.google.com/uc?authuser=0&id=0B2X3Bx1aCHsJR1c0SzNyWlAtZjA&export=download/TileInv10.zip'
```

#### 6.1 A simple function for shapefile extraction

We first build a simple function that ... Download & load shapefile as an `sf` object -- as process.

``` r
get_url_shape <- function (url) {
  temp <- tempdir()
  zip_name <- paste0(temp, '\\', basename(url))
  download.file(url, zip_name, 
                quiet = TRUE)
  unzip(zip_name, exdir = temp)
  x <- sf::st_read(dsn = gsub('\\.zip', '', zip_name), 
                   layer = gsub('\\.zip','', basename(url)),
                   quiet = TRUE) 
  unlink(temp) 
  x}
```

#### 6.2 Hexmap of Congressional districs

Apply function.

``` r
dailykos_shapes <- lapply (c(dailyvos_hex_cd, dailyvos_hex_st), 
                           get_url_shape)
names(dailykos_shapes) <- c('cds', 'states')
#State hex shapefile is slightly broken.
dailykos_shapes$states <- lwgeom::st_make_valid(dailykos_shapes$states)
```

Here, we consider Presidential voting ...

``` r
dailykos_pres_flips <- dailykos_pres_elections %>%
  group_by(District, year) %>%
  filter(percent == max(percent))%>%
  mutate(dups = n()) %>%
  filter(dups != 2) %>% #Kill ties --> n = 3
  select(-percent, -dups) %>%
  spread(year, candidate) %>%
  na.omit()%>%
  mutate(flips = paste0(`2008`, '~',`2012`, '~', `2016`)) %>%
  group_by(flips) %>%
  mutate(sum = n()) %>%
  ungroup()
```

``` r
dailykos_pres_flips %>%
  mutate(f08_12 = paste0(`2008`,'_', `2012`),
         f12_16 = paste0(`2012`,'_', `2016`))%>%
  select(District, f08_12, f12_16) %>%
  gather(elect, flip, -District) %>%
  group_by(elect, flip) %>%
  summarize(value=n()) %>%
  separate(flip, c('source', 'target'), spep = '_') %>%
  separate(elect, c('e1', 'e2'), spep = '_') %>%
  mutate(source = paste0(source, '_', gsub('f','', e1)),
         target = paste0(target, '_', e2)) %>%
  select(-e1, -e2)
```

    ## # A tibble: 8 x 3
    ##   source    target     value
    ##   <chr>     <chr>      <int>
    ## 1 McCain_08 Obama_12       1
    ## 2 McCain_08 Romney_12    191
    ## 3 Obama_08  Obama_12     209
    ## 4 Obama_08  Romney_12     31
    ## 5 Obama_12  Clinton_16   189
    ## 6 Obama_12  Trump_16      21
    ## 7 Romney_12 Clinton_16    15
    ## 8 Romney_12 Trump_16     207

Note that this has been reproduced.

``` r
dailykos_shapes$cds %>%
  inner_join(dailykos_pres_flips)%>%
  ggplot() + 
  geom_sf(aes(fill = reorder(flips, -sum)),
          color = 'gray', 
          alpha = .85) + 
  geom_sf(data=dailykos_shapes$states, 
          fill = NA, 
          show.legend = F, 
          color="black", 
          lwd=.7) +
  ggsflabel::geom_sf_text(data = dailykos_shapes$states,
                          aes(label = STATE), 
                          size = 2.75,
                          color='white',
                          face='bold') +
  ggthemes::scale_fill_stata()+
  theme(axis.title.x=element_blank(),
        axis.text.x=element_blank(),
        axis.title.y=element_blank(),
        axis.text.y=element_blank(),
        legend.title=element_blank(),
        legend.position = 'bottom') +
  labs(title = "Presidential election results - 2008, 2012 & 2016",
       caption = 'Data source: Daily Kos')
```

![](README_files/figure-markdown_github/unnamed-chunk-31-1.png)

#### 6.3 Tile map of US states

Extract shapefiles from ...

``` r
dailykos_tile <- lapply (c(dailyvos_tile_inner,
                           dailyvos_tile_outer),
                         get_url_shape)
names(dailykos_tile) <- c('inner', 'outer')
```

Senators by state by part.

``` r
sens <- csusa_senate_dets %>%
  mutate(party = factor(party, levels =c('democrat', 
                                         'republican', 
                                         'independent'))) %>%
  arrange (state_code, party) %>%
  group_by(state_code) %>%
  mutate(layer = row_number())%>%
  rename(State = state_code) %>%
  select(State, party, layer)
```

Plot. To address the consistency issue. Group by aplhabetical order -- such that if D-R, then D placed in inner. Independents?

``` r
dailykos_tile$outer %>% 
  left_join(sens %>% filter (layer == 2)) %>%
  ggplot() + 
  geom_sf(aes(fill = party),
          color = 'black', 
          alpha = .85) + 
  geom_sf(data = dailykos_tile$inner %>%
            left_join(sens %>% filter (layer == 1)), 
          aes(fill = party)) +
  ggsflabel::geom_sf_text(data = dailykos_tile$inner,
                          aes(label = State), 
                          size = 2.5,
                          color = 'white') +
  ggthemes::scale_fill_stata()+
  theme(axis.title.x=element_blank(),
        axis.text.x=element_blank(),
        axis.title.y=element_blank(),
        axis.text.y=element_blank(),
        legend.title=element_blank(),
        legend.position = 'bottom') +
  labs(title = "115th US Senate Composition by State & Party",
       caption = 'Data sources: Daily Kos & VoteView')
```

![](README_files/figure-markdown_github/unnamed-chunk-34-1.png)

------------------------------------------------------------------------

<iframe id="serviceFrameSend" src="../index.html" width="1000" height="1000"  frameborder="0">
### 7 A work in progress

Indeed, a work in progress. But hopefully a nice round-up of useful open source resources for invewstigating & visualizing federal election results. I would love to here about additional/alternative open source resources!
