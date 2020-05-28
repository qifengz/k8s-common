package controllers

import (
	"errors"
	"fmt"
	"go.uber.org/zap"
	"gopkg.in/src-d/go-git.v4"
	"gopkg.in/src-d/go-git.v4/config"
	"gopkg.in/src-d/go-git.v4/plumbing"
	"gopkg.in/src-d/go-git.v4/plumbing/transport/http"
	"os"
	"strings"
)

func CheckoutTag(gitUrl, localDir, username, password, branch string, tagName string) (string, error) {
	auth := &http.BasicAuth{Username: username, Password: password}

	//from := strings.LastIndex(gitUrl, "/")
	//end := strings.LastIndex(gitUrl, ".")
	//repositoryName := gitUrl[from+1 : end]
	_, err := os.Stat(localDir)
	log.Info("CheckoutTag", zap.String("local dir", localDir))
	var r *git.Repository
	if err != nil {
		if os.IsNotExist(err) {
			err := os.MkdirAll(localDir, 0777)
			if err != nil {
				log.Error("create local dir failed", zap.Error(err))
				return "", err
			}
			cloneOptions := git.CloneOptions{
				URL:               gitUrl,
				RecurseSubmodules: git.DefaultSubmoduleRecursionDepth,
				Auth:              auth,
				Progress:          os.Stdout,
			}
			r, err = git.PlainClone(localDir, false, &cloneOptions)
			if err != nil {
				log.Error("git clone failed", zap.Error(err))
				return "", err
			}

		} else {
			log.Error("stat local dir failed", zap.Error(err))
			return "", err
		}
	} else {
		r, err = git.PlainOpen(localDir)
		if err != nil {
			log.Error("open local dir failed", zap.Error(err), zap.String("repositoryDir", localDir))
			if errors.Is(err, git.ErrRepositoryNotExists) {
				log.Error("invalid local dir")
				os.RemoveAll(localDir)
			}
			return "", err
		}
	}

	tagRefs, err := r.Tags()
	tagRefs.ForEach(func(tagRef *plumbing.Reference) error {
		//log.Info("tagRefs for each", zap.String("tagRefs ", tagRef.Name().String()))
		if tagRef.Name().String() == fmt.Sprintf("refs/tags/%s", tagName) {
			err = r.DeleteTag(tagName)
			return err
		}
		return nil
	})
	if err != nil {
		log.Error("DeleteTag failed", zap.Error(err), zap.String("gitUrl", gitUrl), zap.String("tag", tagName))
	}

	remote, err := r.Remote("origin")
	err = remote.Fetch(&git.FetchOptions{
		Auth:     auth,
		RefSpecs: []config.RefSpec{"+refs/tags/*:refs/rtags/origin/*"},
	})
	if err != nil {
		log.Error("fetch failed", zap.Error(err), zap.String("gitUrl", gitUrl), zap.String("tag", tagName))
	}

	// pull
	/*pullOption := git.PullOptions{
		Auth:       auth,
		RemoteName: "origin",
		//	ReferenceName: plumbing.NewTagReferenceName(tagName),
	}
	pullErr := tree.Pull(&pullOption)
	if pullErr != nil {
		log.Error("pull failed", zap.Error(pullErr), zap.String("gitUrl", gitUrl), zap.String("tag", tagName))
		if !errors.Is(err, git.ErrNonFastForwardUpdate) && !errors.Is(err, git.NoErrAlreadyUpToDate) && !errors.Is(err, plumbing.ErrObjectNotFound) {
			return "", err
		}
	}*/

	ref, err := r.Head()
	//head 对应的d
	commitHash := ref.Hash()
	log.Info("CheckoutTag", zap.String("git pulled and head HEAD is", fmt.Sprintf("%v", commitHash)))

	tree, err := r.Worktree()
	if err != nil {
		log.Error("ch worktree failed", zap.Error(err))
		return "", err
	}
	if tagName != "" {
		// Checkout tag
		log.Info("git checkout tag", zap.String("tag", tagName))
		err = tree.Checkout(&git.CheckoutOptions{
			Branch: plumbing.NewTagReferenceName(tagName),
		})
	} else {
		// Checkout branch
		err = tree.Checkout(&git.CheckoutOptions{
			Branch: plumbing.NewBranchReferenceName(branch),
		})
		log.Info("git checkout branch", zap.String("branch", branch), zap.Error(err))
	}
	if err != nil {
		log.Error("git checkout failed", zap.Error(err))
		return "", err
	}
	//对应的commit id
	headRef, err := r.Head()
	commitHash = headRef.Hash()

	log.Info("CheckoutTag", zap.String("git show-ref --head HEAD after checkout", fmt.Sprintf("%v", commitHash)))
	if err != nil {
		return "", err
	}
	return fmt.Sprintf("%v", commitHash), nil
}

func isHeadTag(localDir string, tag string) (bool, string) {
	_, err := os.Stat(localDir)
	log.Info("isHeadTag", zap.String("local dir", localDir))
	if err != nil {
		log.Error("isHeadTag", zap.Error(err))
		return false, ""
	}
	r, err := git.PlainOpen(localDir)
	if err != nil {
		log.Error("isHeadTag", zap.Error(err))
		return false, ""
	}
	headRef, err := r.Head()
	currentTag, err := getCurrentTagFromRepository(r, headRef.Hash())
	log.Info("isHeadTag", zap.String("currentTag", currentTag), zap.String("Hash", fmt.Sprintf("%v", headRef.Hash())))
	if err != nil {
		log.Error("isHeadTag", zap.Error(err))
		return false, ""
	}
	return currentTag == tag, fmt.Sprintf("%v", headRef.Hash())
}

func getCurrentTagFromRepository(repository *git.Repository, h plumbing.Hash) (string, error) {
	tagRefs, err := repository.Tags()
	if err != nil {
		return "", err
	}
	var currentTagName string
	err = tagRefs.ForEach(func(tagRef *plumbing.Reference) error {
		revision := plumbing.Revision(tagRef.Name().String())
		tagCommitHash, err := repository.ResolveRevision(revision)
		if err != nil {
			return err
		}
		if *tagCommitHash == h {
			currentTagName = tagRef.Name().String()
		}
		return nil
	})
	if err != nil {
		return "", err
	}
	tagName := strings.ReplaceAll(currentTagName, "refs/tags/", "")
	return tagName, nil
}

func getCurrentBranch(localDir string) (string, string, error) {
	_, err := os.Stat(localDir)
	log.Info("getCurrentBranch", zap.String(", local dir", localDir))
	if err != nil {
		log.Error("getCurrentBranch", zap.Error(err))
		return "", "", err
	}
	r, err := git.PlainOpen(localDir)
	if err != nil {
		log.Error("getCurrentBranch", zap.Error(err))
		return "", "", err
	}
	branchRefs, err := r.Branches()
	if err != nil {
		log.Error("getCurrentBranch", zap.Error(err))
		return "", "", err
	}

	headRef, err := r.Head()
	if err != nil {
		log.Error("getCurrentBranch", zap.Error(err))
		return "", "", err
	}
	log.Info("getCurrentBranch", zap.String("head hash", fmt.Sprintf("%v", headRef.Hash())))
	var currentBranchName string
	err = branchRefs.ForEach(func(branchRef *plumbing.Reference) error {
		log.Info("getCurrentBranch", zap.String("branch", branchRef.Name().String()), zap.String("hash", fmt.Sprintf("%v", branchRef.Hash())))
		if branchRef.Hash() == headRef.Hash() {
			currentBranchName = branchRef.Name().String()
			return nil
		}
		return nil
	})
	if err != nil {
		log.Error("getCurrentBranch", zap.Error(err))
		return "", "", err
	}
	branchName := strings.ReplaceAll(currentBranchName, "refs/heads/", "")
	log.Info("getCurrentBranch", zap.String("currentBranchName", branchName))
	return branchName, fmt.Sprintf("%v", headRef.Hash()), nil
}
