---
title: "SimCity 백업 프로그램을 Go로 포팅하기"
date: 2020-10-19T20:44:49+09:00
tags: [Go, Golang, SimCity]
draft: false
categories: [Go]
comments: true
---

어릴적부터 도시를 짓는 게임을 좋아했습니다. 예전에 심시티 3000을 제 돈 주고 산 걸로 시작해서, 심시티 4를 거쳐 지금은 시티즈 스카이라인을 가끔씩 플레이합니다. 

예전에 쓰던 컴퓨터에서 심시티 4를 돌릴 때 외부에서 받은 건물이나, 제가 만든 도시, 그리고 스크린샷을 백업하는 프로그램을 파이썬으로 만든 적이 있었습니다. 

그러다가 이 프로그램을 Go로 포팅하는 일을 해 보고 싶어서 시도해 보게 되었습니다. 

파일 목록을 검색하고 파일을 쓰고 읽는 부분들이 많아서, 개인적으로는 해 볼 만한 내용이었다고 생각합니다. 

그러면 어떤 차이가 있는지 한 번 살펴보도록 하겠습니다. 

# 프로그램 실행 시 매개변수 처리하기

## Python

Python에서는 [argparse](https://docs.python.org/ko/3/library/argparse.html)라는 모듈이 있습니다. 다음과 같이 처리했었습니다. 

```python
import argparse

# Main routine.
if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("-v", "--version",
                        help="Show version of SC4Backup.",
                        action="store_true")
    parser.add_argument("-y", "--yes",
                        help="Backup all user data without asking you.",
                        action="store_true")

    args = parser.parse_args()

    if args.version:
        print_version()

    # 중간 생략

    if args.yes:
        backup_plugin()
        backup_album()
        backup_region()
    else:
        # 중간 생략
```

## Go

Go에는 [flag](https://golang.org/pkg/flag/)라는 모듈이 있습니다. 이 모듈을 이용해서 처리했습니다. 

```golang
func main() {
	fmt.Println("===============SimCity 4 Backup=================")

	printVersion := flag.Bool("version", false, "Show version of SC4Backup.")
    setAllYes := flag.Bool("yes", false, "Backup all user data without asking you.")
    
    flag.Parse()

	if *printVersion == true {
		fmt.Println("SimCity 4 Backup Version v2.0.0")
		fmt.Println("Release Date: 2018-12-31")
		return
    }
    
    // 이후 내용 생략
```

# 파일 목록 가져오기 & 압축 파일 생성하기

이 프로그램에서는 파일들을 모아 압축하는 경우만 있습니다. 그래서 압축 파일을 생성하는 부분 위주로 작성해 보았습니다. 

## Python

Python에서는 [os](https://docs.python.org/ko/3/library/os.html) 모듈의 walk 함수를 사용하여 특정 경로에 있는 파일 목록을 생성했습니다.

```python
def backup_region():
    """ Backup region files. """
    global region_file_name
    global backup_path

    regions_list = []

    # Check today's region backup exists.
    if os.path.exists(backup_path+region_file_name) is True:
        print("Already backuped today's regions data.")
        return

    # Go to regions_path.
    os.chdir(regions_path)

    # Make files of regions directory list.
    for root, _, names in os.walk("."):
        for name in names:
            regions_list.append(os.path.join(root, name))

    print("Make regions file list ... Done. ", len(regions_list), " files.")
    sys.stdout.write("Making backup : ")

    # Make zip file.
    make_backup_file(region_file_name, regions_list)
    os.rename(region_file_name, backup_path+region_file_name)

    print("Making regions backup ... Done.")
```

`os.walk` 함수는 root, dirnames, filenames를 튜플로 반환합니다. root는 디렉터리 이름, dirnames는 하위 디렉터리 이름, 그리고 filenames는 디렉터리가 아닌 파일의 이름 리스트입니다. 

하위 디렉터리 이름은 딱히 필요 없어서 `_`로 대체했습니다. 

파일의 전체 경로를 얻기 위해 `os.path.join(root, name)`을 호출하는 것을 볼 수 있습니다. 

그리고 [zipfile](https://docs.python.org/ko/3/library/zipfile.html) 모듈을 사용하면 ZIP 포맷의 압축 파일을 만들거나 압축을 풀 수 있습니다. 

```python
import zipfile 

def make_backup_file(file_name, file_list):
    """ Make backup file with zipfile library. """
    printed_percent = 0
    backup_file = zipfile.ZipFile(file_name, "w", zipfile.ZIP_DEFLATED)

    for item in file_list:
        backup_file.write(item)
        # 중간 생략

    sys.stdout.write(' \t\t [Done]\n')
    backup_file.close()
```

## Go

Go에서 압축 파일을 만들기 위해 [archive/zip](https://golang.org/pkg/archive/zip/) 모듈을 이용했습니다. 

그리고 파일 목록을 따로 가져오지 않고, [path/filepath](https://golang.org/pkg/path/filepath/) 모듈의 Walk 함수를 이용해서 파일을 읽고 압축 파일에 쓰는 방식으로 처리했습니다. 

```golang
func makeArchiveFile(source string, target string) error {
	zipfile, err := os.Create(target)
	if err != nil {
		return err
	}
	defer zipfile.Close()

	archive := zip.NewWriter(zipfile)
	defer archive.Close()

	info, err := os.Stat(source)
	if err != nil {
		return nil
	}

	var baseDir string
	if info.IsDir() {
		baseDir = filepath.Base(source)
	}

	filepath.Walk(source, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			fmt.Printf("Error on walking SimCity 4 Plugins Directory (%v)\n", err)
			return err
		}

		header, err := zip.FileInfoHeader(info)
		if err != nil {
			return err
		}

		if baseDir != "" {
			header.Name = filepath.Join(baseDir, strings.TrimPrefix(path, source))
		}

		if info.IsDir() {
			header.Name += "/"
		} else {
			header.Method = zip.Deflate
		}

		writer, err := archive.CreateHeader(header)
		if err != nil {
			return err
		}

		if info.IsDir() {
			return nil
		}

		file, err := os.Open(path)
		if err != nil {
			return err
		}
		defer file.Close()

		_, err = io.Copy(writer, file)
		return err
	})

	return nil
}
```

`filepath.Walk` 함수는 탐색을 시작할 경로와, 이 경로에서 발견한 파일을 처리하는 함수를 인자로 받습니다. 

여기서는 익명 함수로 처리했는데, WalkFunc 타입입니다. 이 함수는 경로를 문자열(string)으로 받고, 파일 정보(os.FileInfo 형), error형을 인자로 받습니다. 이 함수는 error type을 반환합니다. 

디렉터리면 파일 헤더에만 추가하고, 일반 파일은 경우 파일을 열어서 `io.Copy` 함수를 호출합니다. 

# 느낀 점들

최근 파이썬에 Type hint 기능이 지원되기는 하지만, 오랜만에 자료형을 따로 지정해 주어야 하는 언어를 다뤄볼 수 있었습니다.

C나 파이썬을 다 써본 입장에서는, 이 경험을 아래와 같이 요약해 볼 수 있었습니다. 

* Go와 같이 자료형을 따로 지정해 주는 언어는 프로그램 내 어떤 자료형을 사용하는지 명확하게 알 수 있어서 좋았다. 
* 터미널에서 `go run <스크립트 이름>`과 같은 명령을 실행하면 따로 빌드하는 과정 없이 스크립트를 실행해 볼 수 있다는 점이 신기했다. (사실 나중에 알게 되었습니다)
* 빠른 시간 내에 뭔가 보여주기에 파이썬 만한 게 없는 것 같다. (이건 제가 파이썬에 익숙해서 그렇게 느끼는 것일수도 있습니다)
    * 다만, 동시에 동작해야 하는 부분이 많을 것 같으면 Go가 좋다고 생각합니다. 다른 언어에 비해 구현하기 편한 것 같아요.

읽어주셔서 감사합니다.