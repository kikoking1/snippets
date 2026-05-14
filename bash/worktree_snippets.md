# Git Worktree Snippets


## Clone new project

- Create a new empty folder for your project `mkdir myProject && cd myProject`
- Git clone with bare flag `git clone --bare <repo-url> .bare`
- Some bare repo style touch ups:
  - identify where your bare repo files are located `echo "gitdir: ./.bare" > .git`
  - then tell the bare repo to create mapping to origin (it otherwise doesn't) 
  `git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*`
  - Then pull the refs `git fetch`

## Add new Branch or Checkout Existing

- Add new worktree and create new branch `git worktree add -b <branch_name> <dir_name>`
- Checkout existing branch as worktree `git worktree add <dir_name> <branch_name>`
- Remember to do things like `npm install` and `nvm use` for new worktrees

### Get Caught Up With Master

- `git fetch` needs to be done each time
- `git merge origin/master`

### Remove Worktree locally (after you're done with the branch/worktree, PR merged, etc)

- `git worktree remove <dir_name>`
- `git branch -D <branch_name>`


### .zshrc helper abstraction

```bash
function worktree() {
  local action=$1

  if [[ -z "$action" ]]; then
    echo "Usage:"
    echo "  worktree add <branch_name>"
    echo "  worktree add -b <branch_name>"
    echo "  worktree clone <repo-url>"
    return 1
  fi

  case "$action" in
    add)
      local branch_name
      local dir_name
      local is_new_branch=0

      # Check if the -b flag is passed
      if [[ "$2" == "-b" ]]; then
        is_new_branch=1
        branch_name="$3"
      else
        branch_name="$2"
      fi

      # Validate that a branch name was provided
      if [[ -z "$branch_name" ]]; then
        echo "Error: Branch name is required."
        return 1
      fi

      # Replace all forward slashes (/) with underscores (_)
      dir_name="${branch_name//\//_}"

      # Execute the appropriate git worktree command
      if [[ $is_new_branch -eq 1 ]]; then
        echo "Creating new branch '$branch_name' in worktree '$dir_name'..."
        git worktree add -b "$branch_name" "$dir_name"
      else
        echo "Fetching and checking out existing branch '$branch_name' in worktree '$dir_name'..."
        git fetch && git worktree add "$dir_name" "$branch_name"
      fi

      # Check if the git worktree command succeeded before proceeding
      if [[ $? -ne 0 ]]; then
        echo "Error: Failed to setup worktree."
        return 1
      fi

      # Navigate into the new worktree directory
      cd "$dir_name" || return 1

      # Check for .nvmrc and execute nvm use if present
      if [[ -f ".nvmrc" ]]; then
        echo "Found .nvmrc, loading node version..."
        nvm use
      fi
      ;;

    clone)
      local repo_url="$2"

      if [[ -z "$repo_url" ]]; then
        echo "Error: Repository URL is required."
        return 1
      fi

      # Extract repo name from URL (removes path up to last slash and .git extension)
      local repo_name
      repo_name=$(basename "$repo_url" .git)

      echo "Setting up bare repository worktree for '$repo_name'..."

      # Create directory and step into it
      mkdir -p "$repo_name" || return 1
      cd "$repo_name" || return 1

      # Clone bare repo
      git clone --bare "$repo_url" .bare

      if [[ $? -ne 0 ]]; then
        echo "Error: Failed to clone repository."
        # Cleanup on failure
        cd ..
        rm -rf "$repo_name"
        return 1
      fi

      # Configure the .git file and remote fetch settings
      echo "gitdir: ./.bare" > .git
      git config remote.origin.fetch "+refs/heads/*:refs/remotes/origin/*"

      echo "Fetching remote branches..."
      git fetch

      echo "Successfully set up $repo_name!"
      ;;

    remove)
      local branch_name="$2"
      local dir_name

      if [[ -z "$branch_name" ]]; then
        echo "Error: Branch name is required."
        return 1
      fi

      # Replace all forward slashes (/) with underscores (_) to get the directory name
      dir_name="${branch_name//\//_}"

      echo "Removing worktree directory '$dir_name'..."
      git worktree remove "$dir_name"

      if [[ $? -ne 0 ]]; then
        echo "Error: Failed to remove worktree '$dir_name'. It might not exist or have uncommitted changes."
        return 1
      fi

      echo "Deleting branch '$branch_name' locally..."
      git branch -D "$branch_name"
      ;;


    merge)
      local branch_name="$2"

      if [[ -z "$branch_name" ]]; then
        echo "Error: Branch name is required."
        return 1
      fi

      echo "Fetching latest changes..."
      git fetch

      if [[ $? -ne 0 ]]; then
        echo "Error: Failed to fetch from remote."
        return 1
      fi

      echo "Merging origin/$branch_name into current branch..."
      git merge "origin/$branch_name"
      ;;

    checkout)
      local target="$2"

      if [[ -z "$target" ]]; then
        echo "Error: Branch name or folder name is required."
        return 1
      fi

      # 1. Remove trailing slash if present (e.g., from tab autocomplete)
      target="${target%/}"

      # 2. Convert slashes to underscores.
      # If they passed a branch name, it becomes the folder name.
      # If they passed a folder name, it stays the folder name.
      local dir_name="${target//\//_}"

      local start_dir="$PWD"
      local current_dir="$PWD"
      local repo_root=""

      # Climb directories looking for .bare
      while [[ "$current_dir" != "/" ]]; do
        if [[ -d "$current_dir/.bare" ]]; then
          repo_root="$current_dir"
          break
        fi
        current_dir=$(dirname "$current_dir")
      done

      if [[ -z "$repo_root" ]] && [[ -d "/.bare" ]]; then
        repo_root="/"
      fi

      if [[ -z "$repo_root" ]]; then
        echo "Error: Could not find a '.bare' repository in the current directory or any parent directories."
        return 1
      fi

      cd "$repo_root" || return 1

      if [[ -d "$dir_name" ]]; then
        echo "Navigating to worktree '$dir_name'..."
        cd "$dir_name" || return 1
      else
        echo "Error: Worktree directory '$dir_name' not found."
        echo "Recommendation: Create it first using:"
        echo "  worktree add <branch_name>"
        echo "  or"
        echo "  worktree add -b <branch_name>"
        cd "$start_dir" || return 1
        return 1
      fi

      # Check for .nvmrc and execute nvm use if present
      if [[ -f ".nvmrc" ]]; then
        echo "Found .nvmrc, loading node version..."
        nvm use
      fi
      ;;

    *)
      echo "Error: Unknown action '$action'."
      echo "Usage:"
      echo "  worktree add <branch_name>"
      echo "  worktree add -b <branch_name>"
      echo "  worktree clone <repo-url>"
      return 1
      ;;
  esac
}
```
