# Steps to Sync Old Fork with Newly Restored Repo

## Step 1: Create a New Fork
- Make a new fork of the restored repo. Let's call it **Final_Fork**.

## Step 2: Clone the Old Fork
- Locally clone the **Old Fork** of the deleted repo.

## Step 3: Run the Sync Script
1. Navigate to the locally cloned copy of the **Old Fork**.
2. Run the following script:

   ```
   git branch -a > branch_list.txt

   while IFS= read -r branch; do
       branch=$(echo "$branch" | tr -d '[:space:]')
       branch_name=$(echo "$branch" | sed 's|^remotes/origin/||')

       if [[ $branch == remotes/origin/tags/* ]]; then
           continue
       fi
       
       git checkout -b "$branch_name" "$branch"
   done < branch_list.txt

3. If you also have tags in your fork:

   ```
   while IFS= read -r tag; do
    tag=$(echo "$tag" | tr -d '[:space:]')
  
    if [[ $tag == remotes/origin/tags/* ]]; then
      tag_name=$(basename "$tag") 
      final_tag=origin/tags/"$tag_name" 
  
      git tag "$tag_name" "$final_tag" 
    fi
   done < branch_list.txt

 

## Step 4: Remove the Old Remote
- After successfully converting all the branches, remove the old remote:
   ```
   git remote remove origin

## Step 5: Add the New Remote
- Add the new remote to your Final Fork:
   ```
   git remote add origin Final_Fork_url

## step 6: Force Push Data
- Force push all the current data to the Final_Fork:
   ```
   git push --all --force
