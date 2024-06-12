name: Daily Commit -

# on:
#   push
    
jobs:
  commit:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2

      - name: Add Test Commit
        run: |
            echo "//Test -Commit" >> README.md
            

      - name: Commit and Push Changes
        run: |
          git config --local user.email "chaudharysarthak46@gmail.com"
          git config --local user.name "SarthakChaudhary46"
          echo "https://SarthakChaudhary46:${{ secrets.PAT }}@github.com/SarthakChaudhary46/Private" > ~/.git-credentials
          git add README.md
          git commit --date='2024-06-09' -m 'Test Commit'
          git push
