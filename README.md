# Reading the bluesky network
David Hood

Building a big selection of follower/following connections from a
initial curated list of related people.

Note: At time of writing, early December 2024, this is done with the
development version of the bskyr (0.2.0) package installed from github,
not the (previous) cran version. If you are replicating this and
functions don’t exist, you might want to look into that. I am also using
dplyr for data processing workflow, and duckplyr for managing what will
be the many many similarly organised csv files on disk.

Note: I have various Sys.sleep() statements scattered through the
downloading code. Bluesky has both short term and medium (daily) rate
limits, and if you go all out with enough data you can blow them. I was
happy to just let my RStudio process tick quitely along for a day or so
until done as an easy way of staying within cummulative rates.

``` r
library(bskyr)
library(dplyr)
library(duckplyr)

# I decline to publicly share my bluesky app authentication details
source("../../R/bluesky_auth.R")
```

I also have a “followers” and “following” folder created in the working
directory, because I am going to be separately storing the
followers/following lists for each account.

## Getting the starting lists

Aggregate curated collections of related people (that people understand
these accounts are related so made into a list or starter pack). If you
are a member of a community then an easy option is to look up what lists
prominent people in your community are on with the clearsky.app. You
want the underlying at://did code, but from knowledge of the list
creator, you can use and option like

``` r
bskyr::bs_get_actor_lists("bluesky.handle..of.list.owner")
```

For starter packs, you can search the directory of starter packs at
blueskydirectory.com/starter-packs/all and from the starter pack
description page which that lists the owner use a command like

``` r
bskyr::bs_get_actor_lists("bluesky.handle..of.list.owner")
```

Given a set of relevant lists (in this example nzlists), we might gather
all the members of those lists with

``` r
listmembers <- function(x){
  ismems <- bs_get_list(x, limit=5000)
  Sys.sleep(2)
  return(ismems)
}

listlist <- lapply(nzlists$uri, listmembers)
listdf <- bind_rows(listlist)

listees <- listdf |> 
  select(handle=subject_handle, name=subject_display_name,
         description=subject_description, created=subject_created_at) |> 
  distinct()
```

And the starter pack members from a collection of starter packs (in this
case nzstarts) with

``` r
startermembers <- function(x){
  ismems <- bs_get_starter_pack(x)
  Sys.sleep(2)
  return(ismems)
}
starterlist <- lapply(nzstarts$uri, startermembers)
starterdf <- bind_rows(starterlist)
listlist <- lapply(starterdf$record_list[!is.na(starterdf$record_list)], 
                   listmembers)
listdf <- bind_rows(listlist)
startees <- listdf |> 
  select(handle=subject_handle, name=subject_display_name,
         description=subject_description, created=subject_created_at) |> 
  distinct()
```

You might also can a seed list of searches of people posting particular
keywords or with profile terms. I didn’t, choosing to stick with
currated knowledge of what people put together.

Then we put the results together and clean them up a little (removing
duplicates) to have a seed list

``` r
nzfolk <- bind_rows(listees,startees) |> distinct()
write.csv(nzfolk, file = "nzcore.csv", row.names=FALSE)
```

## Getting followers and followings

Now, we can start gathering the followers and followings of the seed
group, and saving those as csvs to the followers and following folders.

``` r
nzpop <- read.csv("nzcore.csv")
folk <- sort(nzpop$handle) 
# if it stops (we force stop it for some other computer priority), 
# can check alphabetically last saved and then proceed on

staged_get_flws <- function(x){
  if(x == "handle.invalid"){return(0)}
  flws <- tryCatch(
    expr = {
      suppressMessages(suppressWarnings(bs_get_follows(actor=x, limit = 8000)))
    },
    error = function(e){ 
      return(data.frame(V1=numeric()))
    }
  )
  if(nrow(flws)==0){return(0)}
  subsetcols <- flws |> select(handle, name=display_name,description, created=created_at) |> 
    mutate(followed_by=x)
  write.csv(subsetcols, file=paste0("following/",x,".csv"))
  Sys.sleep(6) # think this is more delay to stay under daily rate limits than I need
  return(1)
}

staged_get_flwrs <- function(x){
  if(x == "handle.invalid"){return(0)}
  flwrs <- tryCatch(
    expr = {
      suppressMessages(suppressWarnings(bs_get_followers(actor=x, limit = 10000)))
    },
    error = function(e){ 
      return(data.frame(V1=numeric()))
    }
  )
  if(nrow(flwrs)==0){return(0)}
  subsetcols <- flwrs |> select(handle, name=display_name,description, created=created_at) |> 
    mutate(follower_of=x)
  write.csv(subsetcols, file=paste0("followers/",x,".csv"))
  Sys.sleep(6) # think this is more delay to stay under daily rate limits than I need
  return(1)
}

# as a catch on resuming if stopped check folder contents check already collected in folder
already_collected <- gsub(".csv$","",list.files("following/")) # already collected
shortlist <- folk[!folk %in% already_collected] # get only uncollected
collect <- lapply(shortlist, staged_get_flws)
# same again for followers
already_collected <- gsub(".csv$","",list.files("followers/")) # already collected
shortlist <- folk[!folk %in% already_collected] # get only uncollected
collect <- lapply(shortlist, staged_get_flwrs)
```

Now we shift tactics slightly, to gathering the connections of people
strongly linked to the current group (both following and followers of
the initial group). Then we repeat that building the group.

This involves picking a Dunbar-like number: a minimum number of follower
and following connections with the group that indicates they are part of
the group. I think that the starting number escalates with each sweep as
the size of the group increases as there is more chance of achieving
multiple connections for other reasons. But I have done no maths on
this. Before running a sweep, I just looked through the profile
descriptions that were going to be collected (as it is built from
followers and followings, there descriptions are within the csvs in the
folders) and made a decision on if I should push up the number to
increase the connectivity requirement. I started with a threshold of
thirty when starting with the follows and followings of the core group
(a group of about 650), and had upped it to fifty a few runs later by
the time I thought “I, through manual inspection, cannot make a case to
include these others in the core group” and stopped gathering. I also
have no idea if specific numbers generalise, so I advocate with trying
different Dunbar-like numbers and inspecting those accounts above the
treshold.

As we are now going to be using a lot of identically formatted csvs,
this is where duckDB through duckplyr helps us out.

``` r
dunbarlikenumber = 50

duckbaseflw <- duckplyr_df_from_csv("following/*.csv")
distinct_follows <- duckbaseflw |> 
  filter(handle !="handle.invalid") |> 
  distinct(handle,followed_by) |> 
  count(handle, name = "flwn") |> 
  filter(flwn > dunbarlikenumber)

duckbaseflwrs <- duckplyr_df_from_csv("followers/*.csv")
distinct_followers <- duckbaseflwrs |> 
  filter(handle !="handle.invalid") |> 
  distinct(handle, follower_of)  |> 
  count(handle, name = "flrn") |> 
  filter(flrn > dunbarlikenumber)

newcore <- distinct_follows |> inner_join(distinct_followers, by = "handle")

shortlist <- sort(newcore$handle[!newcore$handle %in% already_collected])
```

    duckplyr: materializing, review details with duckplyr::last_rel()

``` r
bios <- bind_rows(duckbaseflw |> select(handle, name, description) |> distinct(),
                  duckbaseflwrs |> select(handle, name, description) |> distinct()) |> 
  distinct() |> 
  filter(handle %in% shortlist)
```

    duckplyr: materializing, review details with duckplyr::last_rel()
    duckplyr: materializing, review details with duckplyr::last_rel()

Once confident about the group being collected, it is just a repeat of
the earlier collection

``` r
collect <- lapply(shortlist, staged_get_flws)
collect <- lapply(shortlist, staged_get_flwrs)
```

After getting to the point of not collecting, this is where you can use
duckplyr to create a synthetic data table of all the csvs and ask
questions of it.
