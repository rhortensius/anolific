Anonymise the prolific export and psychopy/pavlovia data
================
Ruud Hortensius

This is a short script to anonymise the prolific export and
psychopy/pavlovia data

It loads the data, uses the {anonymizer} package developed by Paul
Hendricks [source](https://github.com/paulhendricks/anonymizer) to salt
and hash the IDs and writes the csv files per ID.

Put the raw data with the prolific ID in the /sourcedata folder, the
anonymised data will be written to the /data folder. You can share the
data in the /data folder (but not in the /sourcedata folder).

    data
    └───sourcedata
    │   │   PARTICIPANT_anthrom_rating_2020-06-26_09h05.00.314.csv
    │   │   PARTICIPANT_anthrom_rating_2020-06-25_17h55.25.604.csv 
    │   │   ...
    │   PARTICIPANT_anthrom_rating_2020-06-26_09h05.00.314.csv
    │   PARTICIPANT_anthrom_rating_2020-06-25_17h55.25.604.csv  
    |   ...

In this example the data are in a /pilot subfolder of the /data folder.
You can remove pilot if there is only one data folder.

#### Load the prolific export file

``` r
DF.demo <- dir_ls("data/pilot/sourcedata", regexp="\\prolific*") %>% 
  map_dfr(read_csv, .id = "source") %>% 
  filter(status == "APPROVED") 
```

#### Load the psychopy/pavlovia files

``` r
DF.exp2 <- dir_ls("data/pilot/sourcedata",regexp="\\PARTICIPANT*") %>% 
  map_dfr(read_csv, .id = "source") %>% 
  rename(participant_id = prolific_id) %>%
  filter(participant_id %in% DF.demo$participant_id) %>% 
  mutate(source = gsub('data/pilot/sourcedata/', '', source))
```

#### Get the list of unique IDs

``` r
demo_sub <- DF.demo %>% select(participant_id) %>% unique()
exp2_sub <- DF.exp2 %>% select(participant_id) %>% unique()
```

#### Use the anonymize function to salt and hash the IDs

``` r
exp2_sub <- exp2_sub %>% 
  mutate(participant_idA = anonymize(participant_id, .seed = 42))
```

#### Combine it with the other dataframes

prolific export:

``` r
DF.demo <- exp2_sub %>% 
  left_join(DF.demo, by = "participant_id") %>% 
  select(-participant_id) %>% 
  rename(participant_id = participant_idA)
```

psychopy/pavlovia files:

``` r
DF.exp2 <- exp2_sub %>% 
  left_join(DF.exp2, by = "participant_id") %>% 
  select(-participant_id) %>% 
  rename(participant_id = participant_idA)
```

#### Write anonymised files to /data folder

prolific export:

``` r
DF.demo %>% write_csv("data/pilot/prolific_export.csv")
```

psychopy/pavlovia files:

``` r
DF.exp2 %>%
    nest(-source) %>%
    pwalk(~write_csv(x = .y, path = paste0("data/pilot/", .x), na = ""))
```
