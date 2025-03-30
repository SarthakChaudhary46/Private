### Migration Script

```
bash migration.sh <SVN repo> <SVN repo name> <GitHub repo>
```

```bash
# 1. Base Installations
apt-get -y update && apt-get -y upgrade
apt-get install -y \
  ruby-rubygems\
  git-core \
  git-svn \
  ruby \
  subversion \
  curl \
  python3-pip\
  xmlstarlet\
  gh # GitHub CLI
gem install svn2git
pip install pandas
pip install lxml
pip install openpyxl


## Exports
SVN_REPO_URL= SVN_URL
SVN_REPO_NAME= Repo_Name
JFROG_ARTIFACTORY= Artifactory_Name
JFROG_ARTIFACTORY_REPO=demo-generic-local
GITHUB_REPO=svn-gh-test
GITHUB_HOST=https://www.github.com
GITHUB_ORG=StatusNeo


## Pre-Reqs
git config --global user.name "CSG_Bot"
git config --global user.email "bot@csg.com"
git config --global init.defaultBranch main

# 2. Create User Mapping
svn log ${SVN_REPO_URL} -q | awk -F '|' '/^r/ {gsub(/ /, "", $2); sub(" $", "", $2); print $2" = "$2" <"$2">"}' | sort -u > users.txt
## Format: <svn username> = fname lname <email>

# 3. Clone SVN repo & Initialize as Git Repository
git svn clone --stdlayout --authors-file=users.txt ${SVN_REPO_URL}
cd ${SVN_REPO_NAME}
git branch -a >doc.txt

#3.5 Making Branches and adding tags

# svn tags to git tags
while IFS= read -r tag; do
  tag=$(echo "$tag" | tr -d '[:space:]')

  if [[ $tag == remotes/origin/tags/* ]]; then
    tag_name=$(basename "$tag") 
    f_tag=origin/tags/"$tag_name" 

    git tag "$tag_name" "$f_tag" 
  fi
done < doc.txt

# SVN Branch to GIT Branch
while IFS= read -r branch; do
  branch=$(echo "$branch" | tr -d '[:space:]')

  branch_name=$(echo "$branch" | sed 's|^remotes/origin/||')
  
  if [[ $branch == remotes/origin/tags/* || $branch == remotes/origin/trunk* ]]; then
    continue
  fi
  git checkout -b "$branch_name" "$branch"
done < doc.txt

# 4. Handling Binaries &  Log History
find . -type f ! -path "*/.git/*" | while read -r file; do
    mime_type=$(file --mime-type -b "$file")
    case "$mime_type" in
        application/octet-stream|application/x-dosexec|application/x-dfont)
            binary_name=$(basename "$file")
            jfrog_url="${JFROG_ARTIFACTORY}/artifactory/${JFROG_ARTIFACTORY_REPO}/$binary_name"
            echo "$jfrog_url" > "${file}.url"
            
            # Extract original commit information
            svn_info=$(svn log --xml "$SVN_REPO_URL/$file")
            commit_author=$(echo "$svn_info" | xmlstarlet sel -t -v "//logentry/author" -n)
            commit_date=$(echo "$svn_info" | xmlstarlet sel -t -v "//logentry/date" -n)
            commit_msg=$(echo "$svn_info" | xmlstarlet sel -t -v "//logentry/msg" -n)
            
            # Set git environment variables for commit
            GIT_AUTHOR_NAME=$commit_author
            GIT_AUTHOR_EMAIL="${commit_author}@example.com"  
            GIT_AUTHOR_DATE=$commit_date
            GIT_COMMITTER_NAME=$commit_author
            GIT_COMMITTER_EMAIL="${commit_author}@example.com" 
            GIT_COMMITTER_DATE=$commit_date
            
            export GIT_AUTHOR_NAME GIT_AUTHOR_EMAIL GIT_AUTHOR_DATE GIT_COMMITTER_NAME GIT_COMMITTER_EMAIL GIT_COMMITTER_DATE
            
            # Commit the .url file with the original date
            git add "${file}.url"
            git commit -m "$commit_msg"
            # Get the size of the binary file
            binary_size=$(stat -c%s "$file")
            echo "$jfrog_url, $binary_size bytes" >> binary_urls.txt
            
            # Add the size to the total size
            total_size=$((total_size + binary_size))
            ;;
    esac
done

# Add the total size information to the binary_urls.txt file
echo "Total size of binary files: $total_size bytes" >> binary_urls.txt


# 5. Read each JFrog URL from binary_urls.txt
while read -r jfrog_url; do
    # Extract the binary name from the JFrog URL
    binary_name=$(basename "$jfrog_url")

    # Find the path of the binary with the given name
    binary_path=$(find . -name "$binary_name")

    # Upload the binary file to JFrog using curl
    if ! curl -u "${Jfrog_Email}:${Jfrog_Pass}" --upload-file "$binary_path" "$jfrog_url"; then
        echo "Failed to upload $binary_path to JFrog"
        exit 1
    fi
done < binary_urls.txt

# 6 [ Optional ] To remove bin files
 git rm '*.bin'

# 7. Create Excel history & GitIgnore

svn log --xml ${SVN_REPO_URL} > svn_history.xml

echo -e 'import pandas as pd\nfrom openpyxl import load_workbook\nfrom openpyxl.chart import BarChart, Reference\n\n# Load XML data into DataFrame\nxml_data = pd.read_xml("svn_history.xml")\n\n# Convert DataFrame to Excel\nxml_data.to_excel("svn_history.xlsx", index=False)\n\n# Load the Excel file\nwb = load_workbook("svn_history.xlsx")\nws = wb.active\n\n# Calculate total commits and unique users\ncommit_count = len(xml_data)\nunique_users = len(xml_data["author"].unique())\n\n# Append the new information\nws.append(["", "Total Commits", commit_count])\nws.append(["", "Unique Users", unique_users])\n\n# Create a new sheet for the chart\nchart_sheet = wb.create_sheet(title="Histogram")\n\n# Prepare data for the chart\ndata = [\n    ["Metrics", "Value"],\n    ["Total Commits", commit_count],\n    ["Unique Users", unique_users]\n]\n\n# Write data to the chart sheet\nfor row in data:\n    chart_sheet.append(row)\n\n# Create a bar chart\nchart = BarChart()\nchart.type = "col"\nchart.style = 10\nchart.title = "Commit Metrics"\nchart.y_axis.title = "Count"\nchart.x_axis.title = "Metrics"\n\n# Define data for the chart\ndata = Reference(chart_sheet, min_col=2, min_row=1, max_row=3, max_col=2)\ncategories = Reference(chart_sheet, min_col=1, min_row=2, max_row=3)\n\n# Add data to the chart\nchart.add_data(data, titles_from_data=True)\nchart.set_categories(categories)\n\n# Add the chart to the chart sheet\nchart_sheet.add_chart(chart, "D2")\n\n# Save the updated Excel file\nwb.save("svn_history.xlsx")' > convert_to_excel.py

python3 convert_to_excel.py
echo -e '*.py\n*.xml\n!*.xlsx' > .gitignore

git add .
git commit -m "Migration Successfull { By: StatusNeo }"

# 8. Create GH Repo with same name as SVN with GH APIs
gh auth login
gh repo create $GITHUB_REPO --public
git remote add origin ${GITHUB_HOST}/${GITHUB_ORG}/${GITHUB_REPO}
git push --all
git push --tags
```

