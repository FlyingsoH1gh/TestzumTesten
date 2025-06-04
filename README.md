# TestzumTesten


AppMenu.tsx (aus suite-base/src/components/AppBar)
// SPDX-FileCopyrightText: Copyright (C) 2023-2025 Bayerische Motoren Werke Aktiengesellschaft (BMW AG)<lichtblick@bmwgroup.com>
// SPDX-License-Identifier: MPL-2.0

// This Source Code Form is subject to the terms of the Mozilla Public
// License, v2.0. If a copy of the MPL was not distributed with this
// file, You can obtain one at http://mozilla.org/MPL/2.0/

import { Alert, Menu, PaperProps, Snackbar } from "@mui/material";
import { useCallback, useMemo, useState } from "react";
import { useTranslation } from "react-i18next";

import { useStyles } from "@lichtblick/suite-base/components/AppBar/AppMenu.style";
import TextMiddleTruncate from "@lichtblick/suite-base/components/TextMiddleTruncate";
import { LICHTBLICK_DOCUMENTATION_LINK } from "@lichtblick/suite-base/constants/documentation";
import { usePlayerSelection } from "@lichtblick/suite-base/context/PlayerSelectionContext";
import {
  WorkspaceContextStore,
  useWorkspaceStore,
} from "@lichtblick/suite-base/context/Workspace/WorkspaceContext";
import { useWorkspaceActions } from "@lichtblick/suite-base/context/Workspace/useWorkspaceActions";
import { useLayoutTransfer } from "@lichtblick/suite-base/hooks/useLayoutTransfer";
import { IdbExtensionLoader } from "@lichtblick/suite-base/services/IdbExtensionLoader";
import { IdbExtensionStorage } from "@lichtblick/suite-base/services/IdbExtensionStorage";
import { formatKeyboardShortcut } from "@lichtblick/suite-base/util/formatKeyboardShortcut";

import { NestedMenuItem } from "./NestedMenuItem";
import { AppBarMenuItem, AppMenuProps } from "./types";

const selectLeftSidebarOpen = (store: WorkspaceContextStore) => store.sidebars.left.open;
const selectRightSidebarOpen = (store: WorkspaceContextStore) => store.sidebars.right.open;

export function AppMenu(props: AppMenuProps): React.JSX.Element {
  const { open, handleClose, anchorEl, anchorReference, anchorPosition, disablePortal } = props;
  const { classes } = useStyles();
  const { t } = useTranslation("appBar");

  const [nestedMenu, setNestedMenu] = useState<string | undefined>();
  const localLoader = new IdbExtensionLoader("local");
  const storage = new IdbExtensionStorage("local");
  const [snackbarOpen, setSnackbarOpen] = useState(false);
  const [snackbarMessage, setSnackbarMessage] = useState("");
  const [snackbarSeverity, setSnackbarSeverity] = useState<"success" | "error">("success");

  const handleFileUpload = async (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) {
      setSnackbarMessage("Keine Datei ausgewählt.");
      setSnackbarSeverity("error");
      setSnackbarOpen(true);
      return;
    }

    try {
      const arrayBuffer = await file.arrayBuffer();
      const data = new Uint8Array(arrayBuffer);

      // Installation abwarten und den Promise behandeln:
      const extensionInfo = await localLoader.installExtension(data);

      await storage.put({
        content: data, 
        info: extensionInfo, 
      });

      // eslint-disable-next-line no-restricted-syntax
      console.log("Erfolgreich installiert:", extensionInfo);

      setSnackbarMessage(`Erweiterung "${file.name}" erfolgreich installiert!`);
      setSnackbarSeverity("success");
      setSnackbarOpen(true);
    } catch (error) {
      console.error("Fehler beim Installieren der Extension:", error);
      setSnackbarMessage(
        `Fehler beim Installieren: ${
          error instanceof Error ? error.message : "Unbekannter Fehler"
        }`,
      );
      setSnackbarSeverity("error");
      setSnackbarOpen(true);
    }
  };

  /*const handleFileUpload = useCallback((event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) {
      setSnackbarMessage("Keine Datei ausgewählt.");
      setSnackbarSeverity("error");
      setSnackbarOpen(true);
      return;
    }

    const reader = new FileReader();
    reader.onload = async (e) => {
      const arrayBuffer = e.target?.result as ArrayBuffer;
      const data = new Uint8Array(arrayBuffer);

      
      // eslint-disable-next-line no-restricted-syntax
      console.log("Hochgeladene Datei:", file.name, data);
      localLoader.installExtension(data);
    };
    const extensionInfo = await localLoader.installExtension(data);

      await storage.put({
        content: data, 
        info: extensionInfo,
      });

    reader.readAsArrayBuffer(file);
  }, []);*/

  const { recentSources, selectRecent } = usePlayerSelection();

  const leftSidebarOpen = useWorkspaceStore(selectLeftSidebarOpen);
  const rightSidebarOpen = useWorkspaceStore(selectRightSidebarOpen);
  const { sidebarActions, dialogActions } = useWorkspaceActions();

  const handleNestedMenuClose = useCallback(() => {
    setNestedMenu(undefined);
    handleClose();
  }, [handleClose]);

  const handleItemPointerEnter = useCallback((id: string) => {
    setNestedMenu(id);
  }, []);

  const { importLayout, exportLayout } = useLayoutTransfer();
  // FILE

  const fileItems = useMemo(() => {
    const items: AppBarMenuItem[] = [
      {
        type: "item",
        label: t("uploadExtension"),
        key: "upload-extension",
        dataTestId: "menu-item-upload-extension",
        onClick: () => {
          document.getElementById("upload-extension-input")?.click();
          handleNestedMenuClose();
        },
      },
      { type: "divider" },

      {
        type: "item",
        label: t("open"),
        key: "open",
        dataTestId: "menu-item-open",
        onClick: () => {
          dialogActions.dataSource.open("start");
          handleNestedMenuClose();
        },
      },
      {
        type: "item",
        label: t("openLocalFiles"),
        key: "open-file",
        shortcut: formatKeyboardShortcut("O", ["Meta"]),
        dataTestId: "menu-item-open-local-file",
        onClick: () => {
          handleNestedMenuClose();
          dialogActions.openFile.open().catch((err: unknown) => {
            console.error(err);
          });
        },
      },
      {
        type: "item",
        label: t("openConnection"),
        key: "open-connection",
        shortcut: formatKeyboardShortcut("O", ["Meta", "Shift"]),
        dataTestId: "menu-item-open-connection",
        onClick: () => {
          dialogActions.dataSource.open("connection");
          handleNestedMenuClose();
        },
      },
      { type: "divider" },
      { type: "item", label: t("recentDataSources"), key: "recent-sources", disabled: true },
    ];

    recentSources.slice(0, 5).map((recent) => {
      items.push({
        type: "item",
        key: recent.id,
        onClick: () => {
          selectRecent(recent.id);
          handleNestedMenuClose();
        },
        label: <TextMiddleTruncate text={recent.title} className={classes.truncate} />,
      });
    });

    return items;
  }, [
    classes.truncate,
    dialogActions.dataSource,
    dialogActions.openFile,
    handleNestedMenuClose,
    recentSources,
    selectRecent,
    t,
  ]);

  // VIEW

  const viewItems = useMemo<AppBarMenuItem[]>(
    () => [
      {
        type: "item",
        label: leftSidebarOpen ? t("hideLeftSidebar") : t("showLeftSidebar"),
        key: "left-sidebar",
        shortcut: "[",
        onClick: () => {
          sidebarActions.left.setOpen(!leftSidebarOpen);
          handleNestedMenuClose();
        },
      },
      {
        type: "item",
        label: rightSidebarOpen ? t("hideRightSidebar") : t("showRightSidebar"),
        key: "right-sidebar",
        shortcut: "]",
        onClick: () => {
          sidebarActions.right.setOpen(!rightSidebarOpen);
          handleNestedMenuClose();
        },
      },
      {
        type: "divider",
      },
      {
        type: "item",
        label: t("importLayoutFromFile"),
        key: "import-layout",
        onClick: async () => {
          await importLayout();
          handleNestedMenuClose();
        },
      },
      {
        type: "item",
        label: t("exportLayoutToFile"),
        key: "export-layout",
        onClick: async () => {
          await exportLayout();
          handleNestedMenuClose();
        },
      },
    ],
    [
      exportLayout,
      handleNestedMenuClose,
      importLayout,
      leftSidebarOpen,
      rightSidebarOpen,
      sidebarActions.left,
      sidebarActions.right,
      t,
    ],
  );

  // HELP

  const onAboutClick = useCallback(() => {
    dialogActions.preferences.open("about");
    handleNestedMenuClose();
  }, [dialogActions.preferences, handleNestedMenuClose]);

  const onDemoClick = useCallback(() => {
    dialogActions.dataSource.open("demo");
    handleNestedMenuClose();
  }, [dialogActions.dataSource, handleNestedMenuClose]);

  const onDocsClick = useCallback(() => {
    window.open(LICHTBLICK_DOCUMENTATION_LINK, "_blank", "noopener,noreferrer");
    handleNestedMenuClose();
  }, [handleNestedMenuClose]);

  const helpItems = useMemo<AppBarMenuItem[]>(
    () => [
      { type: "item", key: "about", label: t("about"), onClick: onAboutClick },
      { type: "divider" },
      { type: "item", key: "docs", label: t("documentation"), onClick: onDocsClick },
      { type: "divider" },
      { type: "item", key: "demo", label: t("exploreSampleData"), onClick: onDemoClick },
    ],
    [onAboutClick, onDemoClick, onDocsClick, t],
  );

  return (
    <>
      <Menu
        anchorEl={anchorEl ?? (ReactNull as unknown as HTMLElement)}
        anchorReference={anchorReference}
        anchorPosition={anchorPosition}
        disablePortal={disablePortal}
        id="app-menu"
        open={open}
        disableAutoFocusItem
        onClose={handleNestedMenuClose}
        MenuListProps={{
          "aria-labelledby": "app-menu-button",
          dense: true,
          className: classes.menuList,
        }}
        slotProps={{
          paper: {
            "data-tourid": "app-menu",
          } as Partial<PaperProps & { "data-tourid"?: string }>,
        }}
      >
        <NestedMenuItem
          onPointerEnter={handleItemPointerEnter}
          items={fileItems}
          open={nestedMenu === "app-menu-file"}
          id="app-menu-file"
        >
          {t("file")}
        </NestedMenuItem>
        <NestedMenuItem
          onPointerEnter={handleItemPointerEnter}
          items={viewItems}
          open={nestedMenu === "app-menu-view"}
          id="app-menu-view"
        >
          {t("view")}
        </NestedMenuItem>
        <NestedMenuItem
          onPointerEnter={handleItemPointerEnter}
          items={helpItems}
          open={nestedMenu === "app-menu-help"}
          id="app-menu-help"
        >
          {t("help")}
        </NestedMenuItem>
      </Menu>
      <input
        id="upload-extension-input"
        type="file"
        accept=".foxe"
        style={{ display: "none" }}
        onChange={handleFileUpload}
      />
      <Snackbar
        open={snackbarOpen}
        autoHideDuration={6000}
        onClose={() => {
          setSnackbarOpen(false);
        }}
      >
        <Alert
          onClose={() => {
            setSnackbarOpen(false);
          }}
          severity={snackbarSeverity}

          //sx={{ width: "2em" }}
        >
          {snackbarMessage}
        </Alert>
      </Snackbar>
    </>
  );
}








Zusätzlich einfügen in AppBar.ts (suite-base/src/i18n/en)

uploadExtension: "uploadExtension",
